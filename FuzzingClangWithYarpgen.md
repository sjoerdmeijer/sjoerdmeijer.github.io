# Fuzzing Clang/LLVM with yarpgen

## Introduction

Fuzzing is a well-known testing technique that has been extensively documented
across many fields. Within the LLVM community, there has been a very good
presentation on this topic at the 2024 EuroLLVM developer's meeting; the slides
and accompanying [video](https://www.youtube.com/watch?v=3ewJPWW2A00) are
available online. Titled “Practical Fuzzing for C/C++ Compilers”, this work
introduces the main fuzzing tools and explains the underlying high-level
concepts. While it is practical in the sense that it gives this overview of the
tool landscape, the question remains where do you start if you would like to
start fuzzing Clang/LLVM yourself? This blogpost is trying to answer this by
providing basic hints and tips for Clang/LLVM developers where to start with
this.

This post will also make the case that running the fuzzer is the easy part; the
subsequent steps are far more time-consuming. I will focus in particular on
constructing various minimal reproducers, since if others are inspired to
contribute, we want to encourage high-quality, low-noise issue reports.

It's probably easy to agree that testing is a good thing, but why would a
Clang/LLVM developer be interested in fuzzing besides all the other testing that
is happening upstream/downstream?  For example, if you would like to add or
enable a new pass, see
[this](https://llvm.org/docs/DeveloperPolicy.html#adding-or-enabling-a-new-llvm-pass)
section of the Developer Policy, which reads:

> Correctness: The pass should have no known correctness issues (except global
correctness issues that affect all of LLVM). If an old pass is being enabled
(rather than implementing a new one incrementally), additional due diligence is
required. The pass should be fully reviewed to ensure that it still complies
with current quality standards. Fuzzing with disabled profitability checks may
help gain additional confidence in the implementation.

So, it suggests to run fuzzers, but here are no practical tips how to do this
and this blogpost is trying to (partially) address that.

## Problem Definition

A fuzzer is a test-case generator: it produces a valid program (already a
non‑trivial task), then compiles, links, runs it, and checks the output. Running
the fuzzer itself is relatively straightforward; the difficult and
time-consuming part of fuzzing is everything that happens afterwards, such as
analysis and reduction.

To give an overview of the whole process, the general steps are:
- Run the fuzzer.
- Analyse the results and failures:
  - Check whether each failure is already known; this requires maintaining a
    record of all previously observed issues.
  - If the issue is new to you, search the public bug tracker to see whether it
    is already reported.
- If it's a new issue rebuild the compiler: the problem may have been fixed in the time between
  building the compiler and running the fuzzer, which might span a day or even a
  weekend.
- Create a minimized C/C++ source reproducer (for example, using creduce).
- If the problem is not in the front end, use that C/C++ reproducer to derive an LLVM
  IR test case (for example, with llvm-reduce), which is preferable for middle- or
  back-end issues.
  - Use `llvm-reduce` to create a minimal LLVM IR reproducer,
  - Create a github issue.

This suggests that discovering new bugs may require two separate compiler
builds, two independent code reductions, and checks on multiple systems. Because
of this, full automation is feasible but makes it more difficult to design a
test setup that is not very noisy and produces a lot of false positives.
Automation is out of scope for this blog-post, here we try to give insights in
each of the steps identified above.

Several fuzzers are available for compiler testing, with CSmith and Yarpgen
being among the most popular. In this blog post, I’ll focus on Yarpgen, as it is
specifically designed to expose issues in compiler optimisations, including
loop optimisations.

## Running yarpgen

After downloading yarpgen (git clone https://github.com/intel/yarpgen.git)
building it is as simple as:

```
mkdir build
cd build
cmake ..
make -j
```

as also mentioned in its [instructions](https://github.com/intel/yarpgen?tab=readme-ov-file#building-and-running).

To run yarpgen, we’ll use the `run_gen.py script`. Before doing so, we need to
specify which C/C++ compiler to test and how it should be configured. If you
want to test your own build of the LLVM compiler, edit the `scripts/test_sets.txt`
file and update
[this](https://github.com/intel/yarpgen/blob/e0f63b6580cc9f1c50d18928a4273106dc3a6c41/scripts/test_sets.txt#L32)
section by adding something similar to the following:

```
# Spec name | C++ executable name | C executable name | Common arguments                                            | Arch prefix

clang       | /path/to/llvm-project/build/bin/clang++ | /path/to/llvm-project/build/bin/clang | -w  -fPIC |
```

and more edit is required in this file around [this](https://github.com/intel/yarpgen/blob/e0f63b6580cc9f1c50d18928a4273106dc3a6c41/scripts/test_sets.txt#L56) line. Comment out everything, then
add the configuration that you would like to test, for example:

```
clang_o2          | clang     | -O2        |  grace |
clang_o3          | clang     | -O3 -fexperimental-loop-fusion -floop-interchange | grace |
```

Here's an example how to run yarpgen:

```
/path/to/yarpgen $ python3 scripts/run_gen.py -j 100 -t 1440 -o /data/yarpgen.results.10.01.26/
```

This runs yarpgen with 100 threads (`-j 100`) for 24 hours (`-t 1440`) and output all its
results to directory `/data/yarpgen.results.10.01.26/`.

Yarpgen and its generated programs can consume a significant amount of memory.
If many test cases are being aborted due to out-of-memory errors, you can reduce
certain hard-code global settings in Yarpgen's sources, then recompile yarpgen
for the changes to take effect, for example:

```
diff --git a/src/gen_policy.cpp b/src/gen_policy.cpp
index 37cdf1f..55a8cfb 100644
--- a/src/gen_policy.cpp
+++ b/src/gen_policy.cpp
@@ -33,7 +33,7 @@ static void shuffleProbProxy(std::vector<Probability<T>> &vec) {
 GenPolicy::GenPolicy() {
     Options &options = Options::getInstance();

-    stmt_num_lim = 1000;
+    stmt_num_lim = 50;

     loop_seq_num_lim = 4;
     uniformProbFromMax(loop_seq_num_distr, loop_seq_num_lim, 1);
@@ -351,7 +351,7 @@ GenPolicy::GenPolicy() {
     stencil_dim_num_distr.emplace_back(4, 10);
     shuffleProbProxy(stencil_dim_num_distr);

-    array_dims_num_limit = 7;
+    array_dims_num_limit = 4;
     // It looks like ISPC has trouble allocating arrays that require a lot of
     // memory. We limit the number of dimensions to 4.
     // The issue is that we have to always allocate arrays that contain at
```

## Testing strategy

One question we haven't yet addressed is what exactly we want to test.
Evaluating the standard optimisation levels, such as -O2 and -O3, is a good
starting point, but there are many other options and configurations that
exercise code paths not typically explored with the default settings,
potentially revealing new issues. Likewise, different types of compiler builds
may expose additional problems. In the sections below, we discuss both aspects
and provide some examples.

### LLVM configuration

How we configure LLVM (with CMake) matters.  By default, the build type is
Release, which does not enable assertions. There are several trade-offs to
consider. The simplest option is to create a debug build
(`-DCMAKE_BUILD_TYPE=Debug`), which enables assertions but results in a slower
compiler. Alternatively, you can create a release build with assertions enabled
by setting the following CMake variables: `-DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_ASSERTIONS=ON`.
Additional runtime checks can be enabled with
`-DLLVM_ENABLE_EXPENSIVE_CHECKS=ON`, though this increases compilation time.

### Compiler options

The compiler has a lot of options and flags that one can set on the
command-line, this creates a huge test search space. Most testing is probably
done with only a few compiler options, most likely only the optimisation level
and the CPU target. Adding more options is interesting from a testing point of
view, because it can lead to changes in the IR or code that developers weren't
expecting. Here are some tips & tricks that one can consider to let the
optimisation trigger more or expose it to more LLVM IR sequences:
- Add more compiler flags,
- Enable optimisations that are off by default,
- Force an optimisation, e.g. by letting it ignore its cost-model.

Here's is one example:

```
 -O3 -fglobal-isel -floop-interchange -mllvm -loop-interchange-profitabilities=ignore
```

This configuration enables the Global ISel instruction selector, which is
typically disabled by default at -O3. It also activates loop interchange and
instructs the pass to ignore its cost model and profitability analysis. As a
result, unprofitable loop‑interchange candidates are no longer filtered out,
causing more interchanges to occur and increasing the likelihood of uncovering
new issues.

## Analysing Results

In this blogpost we will focus only on the compile time failures. I.e., if you
do an `ls` in your `result/clang` directory, you'll see something like this:

```
data/yarpgen.24.01.26/result/clang$ ls
./  ../  compfail/  compfail_timeout/  miscompare/
```

In this section, we focus only on the failures found in the compfail directory.
For instance, one of the recent test runs produced the following failure from
that directory:

```
...
=== Build log ======================================================
/local/home/smeijer/llvm-project/build_expensive_checks/bin/clang++  -std=c++11 -w  -fPIC -O0 -mcpu=grace -o clang_o2_driver.o -c driver.cpp
/local/home/smeijer/llvm-project/build_expensive_checks/bin/clang++  -std=c++11 -w  -fPIC -O2 -mcpu=grace -o clang_o2_func.o -c func.cpp
=== Build err ======================================================
clang++: /local/home/smeijer/llvm-project/llvm/lib/Target/AArch64/AArch64ISelLowering.cpp:27779: SDValue performSignExtendInRegCombine(SDNode *, TargetLowering::DAGCombinerInfo &, SelectionDAG &): Assertion `(EltTy == MVT::i8 || EltTy == MVT::i16 || EltTy == MVT::i32) && "Sign extending from an invalid type"' failed.
PLEASE submit a bug report to https://github.com/llvm/llvm-project/issues/ and include the crash backtrace, preprocessed source, and associated run script.
Stack dump:
...
```

### CReduce

We need to create a minimal version of `func.cpp` that reproduces the same issue.
A problem report requires such a minimal reproducer. To generate it, we use
creduce, which needs an "interestingness" test script capable of reproducing the
compile error. In some cases, the script can be as simple as the example below.
The compile command is taken directly from the build error log, with its
output redirected to a file using `&> output.txt`. The subsequent `grep` command
then searches that file for the specific assertion message:

```
#!/bin/bash

/path/to/llvm-project/build_expensive_checks/bin/clang++  -std=c++11 -w  -fPIC -O2 -mcpu=grace -o clang_o2_func.o -c func.cpp &> output.txt

MSG="Sign extending from an invalid type"
grep  -q "$MSG"  output.txt
exit $?
```

With the interestingness script in place, we can now run `creduce`:

```
creduce interestingness.test.sh func.cpp
```

`yarpgen` produces very large test cases with many loops, statements, and
expressions. When creduce reduces them, the result is usually small and simple,
like this:

```
#include <algorithm>
unsigned short a;
void d(bool b[]) {
#pragma clang loop vectorize_predicate(enable)
  for (int c(-118); c < 21; c += 3)
    a = std::min(a, (unsigned short)(b[c] ? -01 : 0));
}
```

### llvm-reduce

For a middle‑end or back‑end LLVM problem report, we do not need a C/C++ source
reproducer; instead, we require a minimal LLVM IR test case. To obtain this
minimal IR reproducer, we will use `llvm-reduce`, but first we must extract the
full IR from the existing C/C++ reproducer. This is done by taking the same
compile command and appending `-S -emit-llvm` to it, e.g.:

```
/path/to/llvm-project/build_expensive_checks/bin/clang++  -std=c++11 -w  -fPIC -O2 -mcpu=grace func.cpp -S -emit-llvm
```

Compile that with `llc` and check whether we get the same assert, e.g.:

```
/path/to/llvm-project/build_expensive_checks/bin/llc func.ll
```

After verifying this, we need an llvm-reduce interestingness test script that
reproduces the problem, similar to what we use with `creduce`. The approach is
the same: we run the compile command and redirect the output to a file. One
important difference from creduce is that the input file name is taken from the
script’s argument list, so it should be referenced as `$1` rather than using a
hardcoded file name like `func.ll`:

```
/path/to/llvm-project/build_expensive_checks/bin/llc $1 &> output.txt

MSG="Sign extending from an invalid type"
grep -q "$MSG" output.txt
exit $?
```
Now, we can invoke llvm-reduce and provide it with our interestingness tests
and the IR input file, e.g.:

```
/path/to/llvm-project/build/bin/llvm-reduce --test ./llvm-reduce-interesting.sh func.ll
```

Once it is done and was able to reduce the file, it will print:

```
Done reducing! Reduced testcase: reduced.ll
```

And when everything worked, openeing reduced.ll could see an IR as small as this:

```
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128-Fn32"
target triple = "aarch64-unknown-linux-gnu"

define <vscale x 16 x i16> @_Z1dPb(<vscale x 16 x i8> %wide.masked.gather) #0 {
entry:
  %0 = trunc <vscale x 16 x i8> %wide.masked.gather to <vscale x 16 x i1>
  %1 = sext <vscale x 16 x i1> %0 to <vscale x 16 x i16>
  ret <vscale x 16 x i16> %1
}

attributes #0 = { "target-cpu"="grace" }
```

### Raise a GitHub Issue

We now have all the necessary information to file a problem report and open a
GitHub issue. As an example, I recently created the following issue, which
includes all of the examples and reproducers shown above:
https://github.com/llvm/llvm-project/issues/177925.

## Conclusion

Fuzzing Clang/LLVM with yarpgen has uncovered a large number of issues. I have
reported many of them; while their severity varies, each one has been confirmed
as a genuine defect in the LLVM code base. Some quick statistics:
- I have raised over 60 issues in the last year,
- Most of them, 51, got fixed and are closed already.
- Most of these issues were raised agains loop optimisations and the AArch64
  back-end.
- The LLVM community has been excellent in addressing these issues. I’d also
  like to think that the approach of creating minimal reproducers, as described
  in this post, has helped facilitate their quick and effective responses.

My setup is semi-automated: most steps are automated, but some manual work
remains, such as checking the issue tracker for existing reports. I would like
to move to a fully automated workflow, but building a system that consistently
produces low-noise reports would be a big project.
