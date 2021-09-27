---
title: "Fixing a bug in the Go compiler as a newbie: a deep dive (II)"
heading: "Fixing a bug in the Go compiler as a newbie: a deep dive (II)"
description: ""
summary: "After investigating the nature of the bug and understanding what is happening, we need now to come up with a way to fix it. This post will walk through the steps I took to solve the bug."
slug: optimizing-go-compiler-2
date: 2021-09-24T12:02:00-05:00
categories:
    - "go"
author: Alejandro García Montoro
github: agarciamontoro
community: alejandro.garcia
---

This is the second and last part of a series of posts on fixing an issue in the Go compiler. Make sure you've read [the first](/blog/optimizing-go-compiler-1) part before continuing.

In the first part of the series, we discussed what the issue was, understanding exactly what was happening. Let's use all that knowledge to finally find a fix for the issue.

# A failing test

As we learned in the [SSA post](/blog/ssa-rewrite-rules/#connecting-it-all-together), running the test suite in the Go compiler is as easy as running the following from the Go repository:

```sh
cd src/
./all.bash
```

This will run all tests, which is always nice, but we can also decide to only run the ones in a single file. For example, the file [`test/codegen/memcombine.go`](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/test/codegen/memcombine.go) contains tests checking memory loads and stores. We can run that single file with the following command:

```sh
cd test/
go run run.go -- codegen/memcombine.go
```

And why is this important? Well, because before trying to fix the issue, we need to have a failing test in order to validate whether our proposed solution is correct.

The tests in that file are simple functions containing specially formatted comments that tell the test system what to look out for in the generated code. So, if we don't want the generated code to contain certain instructions in a specific architecture, we can write a comment like the following:

```go
// amd64:-`MOV[BWL]`
```

If a function includes that line, the test will fail when the generated code for that function in the `amd64` architecture contains any of the instructions `MOVB`, `MOVW` or `MOVL`.

As this is exactly what we want to avoid in our function, so we can add the following test to the file, which is our original function with the comment added, and a more descriptive name:

```go
func store_le64_load(b []byte, x *[8]byte) {
	_ = b[8]
	// amd64:-`MOV[BWL]`
	binary.LittleEndian.PutUint64(b, binary.LittleEndian.Uint64(x[:]))
}
```

When running

```sh
go run run.go -- codegen/memcombine.go
```

that test will fail, and the command will output the generated code that contains such instructions.

We are now prepared to try to find a fix. If the test passes, we'll be confident enough that our solution is indeed correct.

# The fix

We know now that our goal is to make sure that the generated code uses a single `MOVQstore`, instead of the smaller `MOVBstore`, `MOVWstore` and `MOVLstore` instructions. We have two ways of getting there:

1. Trying to understand how the rules in the [generic.rules](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/gen/generic.rules) and [AMD64.rules](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/gen/AMD64.rules) files are applied during the compilation, and trying to fix the conflict that generates the weird pattern by adjusting the existing rules: maybe reordering them, maybe making them more specific or more generic, or maybe something else.
2. Adding a new rule that explicitly matches that pattern and changes it with what we need.

My first instinct was to go with 1: It's a more elegant approach, and would let us actually fix the issue, instead of patching it. It would also avoid the overhead of adding a new rule, which should be always checked and only applied when this specific, weird pattern happens.

This proved to be way more difficult than I had anticipated. I did try to deep-dive into the rewriting process by using the following command, which logs every rewrite of every value of the SSA representation.

```sh
go build -gcflags=-d=ssa/lower/debug=2 memcombine.go
```

The output of that command is, let's say, voluminous, and it does not log which rule is being applied each time, only the changes that every value go through. Believe me, trying to read it is not a fun experience.

So I soon abandoned the idea of trying to understand exactly why that weird pattern of smaller `MOV` instructions came to be. And I decided to go with the Good Enough™ approach; i.e., trying to come up with a new rule that matched that specific pattern and converted it to an optimized version.

And so I did it:

```
(MOVBstore [7] p1 (SHRQconst [56] w)
  x1:(MOVWstore [5] p1 (SHRQconst [40] w)
  x2:(MOVLstore [1] p1 (SHRQconst [8] w)
  x3:(MOVBstore p1 w mem))))
  && x1.Uses == 1
  && x2.Uses == 1
  && x3.Uses == 1
  && clobber(x1, x2, x3)
  => (MOVQstore p1 w mem)
```

[That rule](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/gen/AMD64.rules#L1972-L1980) is the literal translation into the rule syntax of our original idea. Let's study, first, how we can come up with such a rule.

Remember that our faulty block looked like this:

```
v171 (+84) = MOVBstore <mem> v248 v34 v1
v168 (+85) = SHRQconst <uint64> [8] v34
v216 (+89) = SHRQconst <uint64> [40] v34
v180 (+91) = SHRQconst <uint64> [56] v34
v219 (88) = MOVLstore <mem> [1] v248 v168 v171
v243 (90) = MOVWstore <mem> [5] v248 v216 v219
v255 (91) = MOVBstore <mem> [7] v248 v180 v243
```

Replacing the uses of the three `SHRQconst` instructions in the last three lines, we have:

```
v171 (+84) = MOVBstore <mem> v248 v34 v1
v219 (88) = MOVLstore <mem> [1] v248 (SHRQconst <uint64> [8] v34) v171
v243 (90) = MOVWstore <mem> [5] v248 (SHRQconst <uint64> [40] v34) v219
v255 (91) = MOVBstore <mem> [7] v248 (SHRQconst <uint64> [56] v34) v243
```

Now, if we replace the usage of the first line in the second one, we arrive at the following:

```
v219 (88) = MOVLstore <mem> [1] v248 (SHRQconst <uint64> [8] v34) (MOVBstore <mem> v248 v34 v1)
v243 (90) = MOVWstore <mem> [5] v248 (SHRQconst <uint64> [40] v34) v219
v255 (91) = MOVBstore <mem> [7] v248 (SHRQconst <uint64> [56] v34) v243
```

Doing the same replacement with `v219:`

```
v243 (90) = MOVWstore <mem> [5] v248 (SHRQconst <uint64> [40] v34) (MOVLstore <mem> [1] v248 (SHRQconst <uint64> [8] v34) (MOVBstore <mem> v248 v34 v1))
v255 (91) = MOVBstore <mem> [7] v248 (SHRQconst <uint64> [56] v34) v243
```

and, lastly, with `v243`:

```
v255 (91) = MOVBstore <mem> [7] v248 (SHRQconst <uint64> [56] v34) (MOVWstore <mem> [5] v248 (SHRQconst <uint64> [40] v34) (MOVLstore <mem> [1] v248 (SHRQconst <uint64> [8] v34) (MOVBstore <mem> v248 v34 v1)))
```

Removing the assignment itself and pretty printing it, we obtain the actual pattern of instructions that we want to fix:

```
MOVBstore <mem> [7] v248 (SHRQconst <uint64> [56] v34)
    (MOVWstore <mem> [5] v248 (SHRQconst <uint64> [40] v34)
    (MOVLstore <mem> [1] v248 (SHRQconst <uint64> [8] v34)
    (MOVBstore <mem> v248 v34 v1)))
```

Instead of this, we want a single `Q`uad instruction that reads from `v34` and stores into `v248`, something like the following:

```
MOVQstore <mem> v248 v34 v1
```

With the actual pattern that we're seeing and the expected instruction that we want to see, we have all the pieces we need to build a new rule. In its simplest form, the rule has the form of `(actual pattern) => (expected pattern)`. In our case, it looks as follows:

```
(MOVBstore [7] p1 (SHRQconst [56] w)
  (MOVWstore [5] p1 (SHRQconst [40] w)
  (MOVLstore [1] p1 (SHRQconst [8] w)
  (MOVBstore p1 w mem))))
  => (MOVQstore p1 w mem)
```

Comparing this with the actual pattern we described before, we can see a couple of changes: We've replaced `v248` with a variable named `p1`, `v34` with a variable named `w`, and `v1` with a variable named `mem` (as it represents the memory). With this renaming, we generalize the possible values those variables can have, but restrict the pattern to be matched only when the same value appears in every instance of the same variable. We've also removed the type specifications from the instructions, as they aren't important in the matching pattern.

The above rule is pretty much identical to the actual rule, which ended up looking like this:

```
(MOVBstore [7] p1 (SHRQconst [56] w)
  x1:(MOVWstore [5] p1 (SHRQconst [40] w)
  x2:(MOVLstore [1] p1 (SHRQconst [8] w)
  x3:(MOVBstore p1 w mem))))
  && x1.Uses == 1
  && x2.Uses == 1
  && x3.Uses == 1
  && clobber(x1, x2, x3)
  => (MOVQstore p1 w mem)
```

The only addition here are the `x1`, `x2`, and `x3` labels, which name each of the values that correspond to the inner `MOV` instructions, and the boolean condition we have at the end of the matching pattern, which is actually literal Go code:

```go
x1.Uses == 1
  && x2.Uses == 1
  && x3.Uses == 1
  && clobber(x1, x2, x3)
```

This boolean condition is simply an additional restriction to the pattern matching. The rules specification says that this pattern will only match if the pattern is matched **and** if this condition is true.

For understanding what the additional restriction is, we need to understand what the `.Uses` field and the `clobber` function do with `x1`, `x2`, and `x3`, which are instances of [the `Value` struct](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/value.go#L17-L63) defined in the compiler. This is actually pretty straightforward:

-   The [`Uses` field](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/value.go#L51-L52) is a field that we can access in every SSA value and, per its definition, is a counter of the number of times that specific value is used.
-   The [`clobber` function](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/rewrite.go#L935-L945) both checks that the value is never used again and decrements the `.Uses` counter.

In short, what we're checking with the boolean condition is that each of the inner `MOV` instructions are used only once, and making sure that they are never used again. This is a simple strategy to restrict the pattern matching to the specific case we want to fix.

# Wrapping up

So... that's it! We did it!

We wrote a failing test:

```go
func store_le64_load(b []byte, x *[8]byte) {
	_ = b[8]
	// amd64:-`MOV[BWL]`
	binary.LittleEndian.PutUint64(b, binary.LittleEndian.Uint64(x[:]))
}
```

And we added a new AMD64 rule:

```
(MOVBstore [7] p1 (SHRQconst [56] w)
  x1:(MOVWstore [5] p1 (SHRQconst [40] w)
  x2:(MOVLstore [1] p1 (SHRQconst [8] w)
  x3:(MOVBstore p1 w mem))))
  && x1.Uses == 1
  && x2.Uses == 1
  && x3.Uses == 1
  && clobber(x1, x2, x3)
  => (MOVQstore p1 w mem)
```

With the addition of this rule, after re-generating the code (check out [the SSA post](/blog/ssa-rewrite-rules/#connecting-it-all-together) to know how to do that) and re-compiling the compiler, the test passes, so we solved the issue!

...or did we? Sorry, there's just one more tiny thing to do. In the issue comments, [@mundaym](https://github.com/mundaym) pointed out [an important thing](https://github.com/golang/go/issues/41663#issuecomment-699904531):

> arm64, ppc64le and s390x all also do unaligned load/store merging.

According to this, if we compile for any of those architectures, using the environment variable `GOARCH` with the values `arm64`, `ppc64le`, or `s390x`, we may see a similar pattern, where the load is done with one instruction but the store is done in multiple pieces.

Long story short, after some investigation, it was easy to see that `arm64` and `ppc64le` were generating correct code, but `s390x` was not.

To cover those four architectures, the test had to get some more assertions in the form of comments:

```go
func store_le64_load(b []byte, x *[8]byte) {
	_ = b[8]
	// amd64:-`MOV[BWL]`
	// arm64:-`MOV[BWH]`
	// ppc64le:-`MOV[BWH]`
	// s390x:-`MOVB`,-`MOV[WH]BR`
	binary.LittleEndian.PutUint64(b, binary.LittleEndian.Uint64(x[:]))
}
```

As you can see, the regex for each of the architectures is different, as the specific instructions differ for each of them.

As I noted before, the only architecture that was not passing the test was `s390x`, so a new rule had to be added. The strategy to do so was exactly the same: Study the pattern of instructions generated by the compiler and craft a rule to specifically match that pattern. [The new rule for S390X](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/gen/S390X.rules#L1423-L1431) ended up taking this shape:

```
(MOVBstore [7] p1 (SRDconst w)
  x1:(MOVHBRstore [5] p1 (SRDconst w)
  x2:(MOVWBRstore [1] p1 (SRDconst w)
  x3:(MOVBstore p1 w mem))))
  && x1.Uses == 1
  && x2.Uses == 1
  && x3.Uses == 1
  && clobber(x1, x2, x3)
  => (MOVDBRstore p1 w mem)
```

The structure is identical to the rule we studied in this series of posts, but if you're interested in the details of the S390X-specific instructions, feel free to dive into [the S390Xops.go file](https://github.com/golang/go/blob/bf48163e8f2b604f3b9e83951e331cd11edd8495/src/cmd/compile/internal/ssa/gen/S390XOps.go) to learn what those instructions do.

# Conclusions

That's it, we did it! This was a long journey to explain a single bug with a 30-line-long fix, but I hope it was an interesting one. At the very least, we learned about:

-   Static Single Assignment (SSA), and how it makes code rewriting and optimizations easier to apply.
-   Rewrite rules, the tool to actually rewrite the code and apply those optimizations.
-   A little bit of assembly, which made it possible to understand the bug and fix it.

Also, the journey made us unveil some particularities of how the Go compiler works and the structure of its code, so I hope you are now even more ready to start contributing!

If you're interested in becoming a Go contributor, go ahead, check the [issue tracker](https://github.com/golang/go/issues?q=is%3Aissue+is%3Aopen+label%3ANeedsFix) and comment on the issue you want to work on. There's no better way to learn how to contribute to a compiler than, well, contributing to a compiler.