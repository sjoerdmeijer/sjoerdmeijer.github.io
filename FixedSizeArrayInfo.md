# Communicating Fixed-Size Array Information from Flang to LLVM IR

*Changelog:*
- 15-05-2026: Initial version.
  - This is a big topic with quite a few moving parts: more updates/revisions
    may follow to make this more complete over time, and entries will be added
    to this changelog to highlight what has been added..

*Disclaimer:*
- This write-up is based on different patches (not necessarily authored by me)
  that are in the review or abandoned state. The aim is to show the
  end-to-end compilation and transformation flow.
- This is orthogonal to the default enablement effort of LoopInterchange in
  Clang and Flang ([PR124911](https://github.com/llvm/llvm-project/pull/124911), [PR140182](https://github.com/llvm/llvm-project/pull/140182), [RFC](https://discourse.llvm.org/t/enabling-loop-interchange/82589)); this is about next steps to handle more cases and our
  motivating examples.

## Introduction

Loop-nest optimisations and DependenceAnalysis (DA) in LLVM can be made more
reliable and accurate by using two pieces of information that are present in
the Fortran source but lost on the way to LLVM IR:

1. The declared shape of fixed-size arrays. This includes stack objects as well
   as globals, where the array type alone is not enough to drive delinearisation
   in an optimal way, i.e. it won't rediscover all array substripts or not
   reliable, which is the goal of Delinearization.
   (we will look at examples later).
2. Tight bounds on loop induction variables that are constrained by the
   language: a Fortran subscript must lie inside the declared bounds of its
   dimension.

We would also like to explore these opportunites for C/C++, but will first
focus on Fortran because the language provides stronger guarantees and our set
of motivating examples includes Fortran codes. This write-up discusses how to
communicate both pieces of information from the Flang front-end to the LLVM
middle-end using two existing IR mechanisms:
- A new `!array.dims` metadata node attached to `alloca` instructions and
  global variable declarations, recording the declared per-dimension extents.
- `llvm.assume` intrinsics emitted at array references, encoding the Fortran
  2018 §9.5.3.3.2 constraint that each subscript lies within its declared
  bounds. The assumes are tagged with `!llvm.array.bounds` so consumers and
  cleanup passes can recognise them.

The middle-end is then taught to consume this information in `Delinearization`,
`ScalarEvolution`, and `DependenceAnalysis`, with a few protective changes in
passes that previously dropped or mishandled these assumes.

## Problem Definition

Two concrete missed opportunities motivate our work, which are defined in P1
and P2 below. Both are blockers for `LoopInterchange` to transform our
motivating examples.

**P1. Delinearization fails for static globals with runtime-variable strides.**

For an access of the form `A(NY*i + j, NY*IL + JL)`, where `A` is a static
global declared as `REAL A(1000,1000)` and `NY` is a runtime value, e.g. a
function argument, parametric delinearisation recovers `NY` (not `1000`) as the
inner dimension and is correctly rejected by Delinearisation's validation step.
Fixed-size delinearisation, in turn, cannot infer dimensions because the
strides themselves contain `NY`, and it has no other source for the declared
shape.  Reproducers of one of our motivating examples have been added as
LoopInterchange regression test:
- [llvm/test/Transforms/LoopInterchange/large-nested-4d.ll](https://github.com/llvm/llvm-project/blob/main/llvm/test/Transforms/LoopInterchange/large-nested-4d.ll)
- [llvm/test/Transforms/LoopInterchange/large-nested-6d.ll](https://github.com/llvm/llvm-project/blob/main/llvm/test/Transforms/LoopInterchange/large-nested-6d.ll)

**P2. The Banerjee MIV test cannot disprove dependences without a tight backedge-taken count.**

Even when delinearisation succeeds, `DependenceAnalysis` falls back to the
"constant-max BTC" when the "symbolic BTC" is not usable, which often evaluates to
`INT_MAX - 1` and that makes Banerjee's coefficient bounds loose and prevents
elimination of infeasible direction vectors. There is currently no in-IR signal
that ties `i` to the array's declared extent.

## Solution Approach

The proposal has three layers: a producer side in Flang, a small IR surface,
and consumer-side changes in the middle-end.

**1. IR surface.**

- `!array.dims`: an integer-list metadata node attached to `alloca` and global
  variable declarations giving the declared extent of each dimension in
  memory-layout order.
- `!llvm.array.bounds`: a marker attached to `llvm.assume` calls that encode an
  array-bounds constraint. The marker enables targeted handling. For example,
  this makes it possible to distinguish it from other assume and could help
  with preservation or deletion of it across different passes.

**2. Producer (Flang).**

- Attach `!array.dims` to globals and to allocas for static arrays.
- During FIR-to-LLVM lowering of `fir.array_coor` for non-boxed arrays with
  known shape, emit two assumes per dimension:
  - `assume(idx >= lb)`
  - `assume(idx <= lb + extent - 1)`
  both tagged with `!llvm.array.bounds`.
  - Merge request: https://github.com/llvm/llvm-project/pull/178811
- A Flang IR pass `RemoveRedundantArrayBoundAssumes` deduplicates these per
  `(predicate, constant-bound)` pair per load, keeping the assume bloat bounded
  while preserving one constraint per induction variable feeding each access.

**3. Consumers (LLVM middle-end).**

- *Delinearization*: when stride-based dimension inference fails, we fall back to
  `!array.dims` on the underlying object. Function
  `validateDelinearizationResult` has been adapted for the case where every
  dimension is known from metadata, i.e. we now use `!array.dims` to extract the
  outer dimension as an extra hint to subscript recognition.
- *DependenceAnalysis*: this has been adapted to try parametric delinearisation
  first as it yields constant-coefficient subscripts that are friendlier to
  Banerjee and then trust `!array.dims` to skip validation in
  `tryDelinearizeFixedSize`. Crucial step in this is that in `collectCoeffInfo`,
  we keep both the symbolic and the constant-max trip count, compute Banerjee
  bounds with each, and pick the tighter total per level — this preserves
  algebraic simplifications such as `NY - (NY-1) = 1` while still benefiting
  from the constant fallback. Use the context-sensitive `isKnownPredicateAt`
  so that an in-scope `assume(NY > 0)` lets a coefficient simplify to `NY`
  instead of `smax(0, NY)`.

- *ScalarEvolution*: communicating extra bound information with @llvm.assume
  intrinsics has the great advantage that SCEV already has support for these
  intrinsics.  For example, SCEV already looks for @llvm.assumes in preheader
  blocks and can thus calculate a tighter BTC. A small addition to function
  `computeBackedgeTakenCount` now walks in-loop body bocks `llvm.assume` calls
  and, for `assume(IV < C)` with `IV` a loop AddRec and `C` loop-invariant,
  derive a constant max BTC of `floor((max(C) - 1 - min(Start)) / Step)`. This
  is the mechanism by which the array-bound assumes tighten the BTC used by
  `DependenceAnalysis`.

- *Protective changes to existing passes*:
  The @llvm.assume intrinsics are attached to all load instructions. These extra
  instructions create new values in the IR and CFG that subsequent passes have to
  process and deal with. Some passes needed to be taught (minor changes) how to
  deal with these intrinsics:
  - `IndVarSimplify::eliminateIVComparison`: do not delete an `assume` of an IV
    against a constant — once SCEV consumes it, it becomes trivially true and
    would otherwise be removed, defeating its purpose.
  - `LoopInterchange`: treat `llvm.assume` as a safe instruction, otherwise
    loops with bound assumes are reported as not tightly nested.
  - `LICM`: do not hoist `llvm.assume` out of the loop where its constraint is
    meaningful for SCEV.

## Motivating Example

We start with the simplest possible case, and then sketch the parametric case that
drives the dependence-analysis side.

### Minimal Fortran

```fortran
subroutine foo()
  real, save :: A(1000)
  integer :: i
  do i = 1, 1000
    A(i) = A(i) + 1.0
  end do
end subroutine
```

### How it gets translated

The global gets `!array.dims` recording the declared shape, and each subscript
reference produces a pair of bounds assumes tagged with `!llvm.array.bounds`:

```llvm
@_QFfooEa = internal global [1000 x float] zeroinitializer, !array.dims !0
..
body:
  store i32 %3, ptr %1, align 4, !tbaa !5
  %7 = load i32, ptr %1, align 4, !tbaa !5
  %8 = sext i32 %7 to i64
  %9 = icmp sge i64 %8, 1
  %10 = icmp sle i64 %8, 1000
  call void @llvm.assume(i1 %9), !llvm.array.bounds !11
  call void @llvm.assume(i1 %10), !llvm.array.bounds !11
  %11 = sub nsw i64 %8, 1
  %12 = mul nsw i64 %11, 1
  %13 = mul nsw i64 %12, 1
  %14 = add nsw i64 %13, 0
  %15 = getelementptr float, ptr @_QFfooEa, i64 %14
  %16 = load float, ptr %15, align 4, !tbaa !12
  %17 = fadd contract float %16, 1.000000e+00
  store float %17, ptr %15, align 4, !tbaa !12
  %18 = load i32, ptr %1, align 4, !tbaa !5
  %19 = add nsw i32 %18, 1
  %20 = sub i64 %4, 1
  br label header
..
!0 = !{i64 1000}
```

What the consumers do with this:

- `Delinearization` reads `!array.dims !{i64 1000}` from `@_QFfooEa` and
  recovers a 1-D shape even when the access pattern alone would be ambiguous.
- `ScalarEvolution::computeBackedgeTakenCount` sees `assume(i <= 1000)` on the
  `i32`/`i64` AddRec `i` and derives a constant max BTC of `999`, instead of
  `INT_MAX - 1` from the type.

### Why this matters: the parametric case

The same mechanism unlocks the harder case that originally motivated the work:

```fortran
subroutine bar(NY, IL, JL)
  integer :: NY, IL, JL, i, j
  real, save :: GlobL(1000, 1000)
  do i = 1, NY
    do j = 1, NY
      GlobL(NY*i + j, NY*IL + JL) = 0.0
    end do
  end do
end subroutine
```

Lowering attaches `!array.dims !{i64 1000, i64 1000}` to `@GlobL` and emits, at
the access, the four bounds assumes `1 <= NY*i + j <= 1000` and `1 <= NY*IL +
JL <= 1000`. With this:

- `Delinearization` recovers the 1000×1000 shape from metadata even though the
  strides contain `NY`.
- The per-IV bound assumes give SCEV a tight constant max BTC for `i` and `j`.
- `DependenceAnalysis` runs Banerjee MIV with both symbolic and constant trip
  counts and picks the tighter total bound per level, allowing it to disprove
  infeasible direction vectors and unblock `LoopInterchange`.

### Pros and Cons

*Cons:*

The obvious challenges with emitting intrinsics and more instructions are:
- increase in compile-time as compiler passes have to process more instructions,
- less optimal codegen as extra instructions can influence heuristics and are
  opaque to the optimisers.

To assess the impact of these intrinsics, I have measured the number of emitted
@llvm.assume intrinsics for different SPEC FP apps and also the perf impact:

| SPEC2017 FP app  | # assume | perf diff |
|------------------|----------|-----------|
| 503.bwaves_r     |  4       | +2%       |
| 507.cactuBSSN_r  | 10       | -         |
| 521.wrf_r        | 1550     | -         |
| 527.cam4_r       | 1526     | -         |
| 549.fotonik3d_r  | 53       | -         |
| 554.roms_r       | 431      | -         |

There seems to be no impact on performance (third column); bwaves is slightly
better, but I haven't analysed if we just get lucky somewhere, or that e.g. a
more precise BTC is really helping somewhere.

The surprising aspect of this experiment is the number of emitted @llvm.assume
intrinsics (second column) because:
- we emit them only for statically declared globals, and
- I added a pass that optimises away redundant constraints.

TODO:
- measure the compile-time impact,
- measure a larger set of benchmarks.


*Mitigations*

To reduce the impact of these @llvm.assume intrinsics, I'd propose to add two
simple IR passes:
- A very early IR pass that makes the @llvm.assume constraints unique. The
  front-end may emit many, and making them unique is probably easier to do
  on IR than at the place where they are emitted in the front-end (although
  it wouldn't hurt to double check this). The idea is that this is a very
  simple pass and scan over the blocks, it keeps a list of seen constraints,
  and drop the ones that are already seen.
- Anoter pass that drops the assumes entirely. This could run just before
  or after vectorization, but after all loop optimisation passes, so that
  the rest of the passes in the pipeline won't have to deal with these
  intrinsics.
- TODO: determine the impact of these passes, and if it really helps.

*Pros:*

- The @llvm.assume intrinsics are "natively" supported by SCEV. It will pick up
  assume intrincis and e.g. use it in backedge-taken calculation.
- The intrinsics help with Delinearisation, but communicating the BTC in this
  form can be picked up by other optimisers as well, and thus be generally
  good thing to do.


### Alternatives

I see two main alternatives to the icmp + @llvm.assume scheme proposed above:
metadata, and operand bundles. The metadata route is briefly mentioned for
completeness; the operand-bundle route is sketched in more detail because it
directly addresses the *Cons* listed above.

#### Convert llvm.assume to Metadata

We could intercept the llvm.assume intrinsics very early in the pipeline, drop
the assumes, and convert the information to a backedge-taken count (BTC)
meta-data node.  This could be a new node `!llvm.loop.btc` or something similar
that we attach to the `!llvm.loop` node. Dropping all assumes very early could
make the approach less compile-time senstive and intrusive.

The main drawback is that LLVM metadata is by contract *droppable*, i.e.  a
transformation is required to drop any metadata attachment that it does not
know or know it can't preserve. This  makes any analysis built on it
best-effort. Going from a guarantee in the forms of assumes to a best-effort
and meta-data is not the robust solution that we are looking for. For example,
it might not be too difficult to imagine that other loop transformations might
change the backedge-taken count.

```llvm
6:
  %7 = sitofp i32 %3 to float
  %8 = sext i32 %3 to i64
  %9 = sub nsw i64 %8, 1
  %10 = getelementptr float, ptr @_QMglobalsEarr, i64 %9
  store float %7, ptr %10, align 4, !tbaa !5
  %11 = add nsw i32 %3, 1
  %12 = sub i64 %4, 1
  br label %2, !llvm.loop !11
..
!11 = distinct !{!11, !12}
!12 = !{!"llvm.loop.btc", i64 99}
```

This meta-data node now contains a new node “btc” which is short for back-edge
taken count, which in this case is 99:

#### Operand bundles

Operand bundles share metadata's lightweight approach but with a much stronger
contract: dropping them is incorrect and will change program semantics (see
[LangRef](https://llvm.org/docs/LangRef.html#id2052)).

There are two ways to apply operand bundles to this proposal. The first is
implementable today; the second might be cleaner but requires an
LLVM IR extension so would be more difficult.

**A. Operand bundles on the existing `@llvm.assume` calls**

Operand bundles are only legal on `call` and `invoke`. However,
the existing `assume_opbundles` mechanism in [LangRef](https://llvm.org/docs/LangRef.html#assume-operand-bundles)
is the standard way to attach *named* facts to an `@llvm.assume(i1 true)`. We
can replace the icmp+assume sequence with a single bundled assume per
access:

```llvm
; 4 instructions per dimension per access:
%lo    = icmp sge i64 %idx, 1
%hi    = icmp sle i64 %idx, 1000
call void @llvm.assume(i1 %lo), !llvm.array.bounds !11
call void @llvm.assume(i1 %hi), !llvm.array.bounds !11

; bundled-assume alternative: 1 instruction, no new SSA values:
call void @llvm.assume(i1 true)
    [ "array.bound"(ptr @_QFfooEa, i64 %idx64, i64 1, i64 1000, i64 0) ]
```

This bundled assume is still an `@llvm.assume` call instruction and it still
costs one call per access in the IR, but could be an attractive solution
because there are no extra SSA values/instructions, cannot be silently dropped,
and reuses existing infrastructure `AssumeBundleQueries` and
`AssumeBundleBuilder`.

This approach was prototyped in this patch:
- https://github.com/llvm/llvm-project/commit/51e0f4fcdef8d213d7af921af728f70140d68032#diff-1d1772820f1edf0653335111c7ab690c53a2df20887eadadf55d7a9c17aa907aR2997

**B. Operand bundles directly on `load` / `store`?**

This is less than a half-baked idea and would required an investigation if this
has any legs at all. The question is, instead of emitting any new intrinsics at
all, could we attach the same information to existiging instructions? This
would be an LLVM IR extension as operand bundles on loads are not yet
supported, the idea is that we attach the access to the load instruction,
something like this:

```llvm
%v = load float, ptr %p, align 4
       [ "array.bound"(i64 %i, i64 1, i64 1000, i64 0),
         "array.bound"(i64 %j, i64 1, i64 1000, i64 1) ],
       !tbaa !12
```

Since this would be an IR extension, which will be intrusive, this direction
has not been further explored.

### Improve DA

Perhaps there are ways to make DependenceAnalysis smarter without any of this
front-end information and @llvm.assumes. This is something that @kasuga-fj was
wondering and prototyped in this patch:
- https://github.com/kasuga-fj/llvm-project/commit/cef19d2804dbc8747e94932f9ff5b6e505979244

This is of course an attractive direction if this were to solve our motivating
examples, but the explicit information is most likely the more reliable
solution for Delinearization and DependenceAnalysis, and could also help other
transformations.
