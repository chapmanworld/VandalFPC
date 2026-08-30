# VandalFPC

**VandalFPC** is a purpose-specific, pinned build of the Free Pascal Compiler used by the **Vandal Engine** game engine and VandalSDK.

It is not intended to replace the Free Pascal project, track every upstream FPC feature, provide a complete alternative RTL, or serve as a general-purpose Pascal distribution.

The purpose of this repository is much narrower:

> Preserve a known compiler state with the language semantics, code generators, and target support required by Vandal Engine, and make that compiler reproducible for VandalSDK builds.

The initial baseline is based on Free Pascal development revision:

```text
FPC version:       3.3.1 development
Git commit:        2ff18e48a0dbc1fa9ba54c0ce9f0bf670ddc7d80
SVN revision:      r46891
Commit date:       2020-09-18
Initial bootstrap: FPC 3.2.2
```

## Why does this repository exist?

Vandal Engine is written using Free Pascal's `DelphiUnicode` language mode and makes extensive use of records, operator overloading, arrays, vectors, matrices, and other mathematical types.

For game-engine mathematics, concise expressions matter.

For example, Vandal wants to be able to express a vector naturally:

```pascal
V := [1.0, 2.0, 3.0];
```

rather than requiring:

```pascal
V := Vector(1.0, 2.0, 3.0);
```

Likewise:

```pascal
V += [1.0, 0.0, 0.0];

V := V + [0.0, 1.0, 0.0];
```

These expressions can be implemented cleanly using static-array types and record operator overloads:

```pascal
type
  TArray3OfSingle = array[0..2] of Single;

  TVector = record
    class operator Implicit(
      const Src: TArray3OfSingle
    ): TVector;
  end;
```

For a game engine containing large amounts of vector, matrix and tensor mathematics, this is not merely cosmetic. It makes mathematical code substantially easier to read and allows source expressions to stay close to the notation used in technical and mathematical references.

## The array-constructor problem

Free Pascal has historically had difficulty distinguishing between Pascal **set constructors** and **array constructors** in overloaded-expression contexts.

The syntax:

```pascal
[1, 2, 3]
```

can represent something set-like in Pascal, but in an appropriate typed context it can also represent an array constructor.

When the compiler resolves the expression incorrectly as a set before considering the expected array type or an overloaded operator, errors such as these result:

```text
Incompatible types: got "Set Of Byte"
```

or, where the elements cannot legally be members of a Pascal set:

```text
Ordinal expression expected
```

For example:

```pascal
V := [0.1, 0.5, 3.0];
```

fails particularly clearly when affected by this problem because `Single` values are not ordinal and therefore cannot form a Pascal set.

## Historical reports

This behavior has appeared in several related FPC/Mantis reports.

### Mantis #34021 — Implicit array in operator is treated like "set of"

This report demonstrated that an array literal supplied to an overloaded operator could be interpreted as `Set Of Byte`.

The issue was fixed in SVN revision **r39554**, represented in the converted Git history by commit:

```text
32c307e9ce8297a2dcd7bca0e4f75b42a7781756
```

That fix was important, but did not cover every operator or conversion path.

### Mantis #35061 — Operator overloads see literal arrays as "Set of ..."

This issue was reported by **Craig Chapman** after finding that the earlier repair did not fully address assignment operators.

The report demonstrated:

```pascal
aRec := [2, 5];
```

being interpreted as:

```text
Set Of Byte
```

instead of being considered as an array value suitable for overloaded assignment or conversion.

The report specifically referenced #34021 and noted that its fix had been limited to particular operator handling.

### Mantis #34526 — := class operator bug with implicit arrays

A closely related report demonstrated the same fundamental problem with an overloaded assignment operator.

It was fixed in SVN revision **r41844**, Git commit:

```text
18519c95
```

That change improved operator and conversion resolution, but it still did not make an array constructor convertible to a **static array** in the case required by Vandal.

Later compilers containing this fix still rejected:

```pascal
V := [0.1, 0.5, 3.0];
```

when the overloaded conversion expected:

```pascal
array[0..2] of Single
```

The compiler still reached set semantics and reported:

```text
Ordinal expression expected
```

### Mantis #36909 — Static array initialization from array constructor

The decisive change was Mantis **#36909**, titled:

> `[PATCH] Static array initialization from array constructor`

This was fixed on **2020-09-18** in SVN revision **r46891**, Git commit:

```text
2ff18e48a0dbc1fa9ba54c0ce9f0bf670ddc7d80
```

The patch added explicit compiler support for conversion from an array constructor to a static array.

Among the compiler changes was the addition of the conversion:

```text
tc_arrayconstructor_2_array
```

This is the behavior Vandal requires.

Testing the compiler at exactly this revision confirms that expressions such as the following compile and execute correctly:

```pascal
V := [0.1, 0.5, 3.0];

V := V + [10.0, 20.0, 30.0];

V := [1.0, 2.0, 3.0] + V;

V += [1.0, 2.0, 3.0];

V -= [0.5, 1.0, 1.5];
```

The C-style compound assignment forms require FPC's `-Sc` compiler option.

## Why pin the compiler?

FPC version number `3.3.1` identifies the long-running development compiler rather than a single released compiler state.

Consequently:

```text
FPC 3.3.1
```

is not sufficiently precise for a reproducible SDK toolchain.

VandalFPC instead identifies the compiler by exact source revision.

The initial VandalFPC baseline is therefore:

```text
2ff18e48a0dbc1fa9ba54c0ce9f0bf670ddc7d80
```

This gives VandalSDK a known compiler whose behavior can be qualified and retained independently of subsequent changes to FPC trunk.

The compiler may move beyond this revision in the future, but only deliberately.

Updates should be driven by Vandal's requirements, such as:

* compiler correctness fixes;
* x86-64 backend fixes;
* AArch64 backend fixes;
* ABI corrections;
* object-file generation fixes;
* new target support;
* linker/toolchain interoperability;
* security or correctness problems affecting the compiler;
* fixes required by VandalSDK itself.

An upstream change is not automatically desirable simply because it is newer.

## Is this an FPC fork?

Technically, yes.

Philosophically, it is better described as a **pinned FPC toolchain for VandalSDK**.

The intention is not to maintain an independently evolving Pascal language implementation.

Where possible, VandalFPC should remain close to its upstream Free Pascal ancestry. Future compiler fixes may be selectively cherry-picked from upstream, with the resulting compiler qualified against Vandal's supported targets.

Large unnecessary changes to the compiler should be avoided.

Similarly, unused compiler modes and subsystems do not need to be removed merely because Vandal does not use them. Keeping the source reasonably close to upstream makes understanding and selectively incorporating upstream fixes considerably easier.

## What Vandal actually needs from FPC

Vandal Engine does **not** require the complete Free Pascal software ecosystem.

In particular, VandalSDK increasingly provides its own low-level runtime facilities, including functionality such as:

* heap management;
* thread management;
* TLS initialization and management;
* platform memory acquisition;
* platform abstraction;
* C-ABI module boundaries.

The engine also deliberately uses C-compatible ABI boundaries where practical so that code generated by different compilers and vendor toolchains can interoperate.

As a result, the critical value provided by FPC to Vandal is primarily:

1. the Pascal language front end;
2. `DelphiUnicode` language semantics;
3. the x86-64 code generator;
4. the AArch64 code generator;
5. correct object-file and ABI generation;
6. a small subset of required RTL units;
7. enough target infrastructure to build those units.

Vandal does not require every FPC package or every piece of the standard RTL.

A custom or reduced `System` integration may eventually be used if target requirements make that worthwhile, but this is not an immediate goal of VandalFPC.

## Target philosophy

VandalSDK is intended to support targets including:

* Windows x86-64;
* Linux x86-64;
* Windows AArch64;
* Linux AArch64;
* Android;
* Steam Deck;
* console platforms where appropriate SDK access is available.

Console SDKs commonly provide platform-specific Clang/LLVM-derived toolchains.

Vandal's preferred model is not necessarily to route Pascal through LLVM.

Instead, wherever possible:

```text
Pascal
   |
   v
VandalFPC
   |
   v
native .o / .obj
   |
   v
small C ABI platform shim
   |
   v
vendor compiler / linker / SDK
```

This keeps platform-specific and potentially restricted SDK interfaces outside the general engine compiler while allowing the vendor's supported C/C++ toolchain to own the final platform boundary.

LLVM support remains potentially useful as an escape hatch for targets that genuinely require LLVM-specific behavior, but it is not intended to be the primary Vandal compilation model.

## Stability policy

VandalFPC does not claim that its pinned compiler is universally more stable than an official FPC release.

It claims something narrower:

> This is the compiler state qualified for building VandalSDK.

Compiler changes should therefore be evaluated against Vandal's requirements rather than against the full set of use cases supported by upstream FPC.

Particular attention should be given to things that cannot easily be corrected after compilation:

* x86-64 code-generation correctness;
* AArch64 code-generation correctness;
* optimizer correctness;
* calling conventions;
* parameter and return-value ABI;
* record and structure layout;
* position-independent code;
* relocation generation;
* atomics;
* TLS code generation;
* exception and unwind information.

Some object-format or linker-facing problems may be manageable elsewhere in VandalSDK. Incorrect machine code is not.

Target qualification is therefore more important to this project than broad compatibility with every FPC RTL package or language mode.

## Regression testing

The array-constructor behavior that motivated this repository should remain covered by an explicit compiler regression test.

At minimum, the test should verify:

```pascal
V := [0.1, 0.5, 3.0];

V := V + [1.0, 2.0, 3.0];

V += [1.0, 2.0, 3.0];
```

This is particularly important because the historical FPC reports show multiple instances of closely related set-versus-array-constructor resolution problems being fixed in different compiler paths.

It would therefore be unwise to assume that one fix permanently guarantees every related operator or conversion scenario.

## Upstream relationship

VandalFPC remains derived from the Free Pascal Compiler.

The existence of this repository should not be interpreted as criticism of the FPC project or its maintainers. Free Pascal supports a very broad collection of language modes, CPUs, operating systems, runtime environments, and compatibility requirements.

Vandal has much narrower priorities.

Those priorities do not necessarily justify changes to upstream FPC, nor should VandalSDK's release schedule depend on the upstream project adopting them.

A pinned toolchain allows both projects to make those decisions independently.

Where VandalFPC discovers a reproducible compiler defect that is relevant to upstream FPC, submitting a focused upstream report and regression test may still be appropriate.

## Contributions and issue tracking

This repository is public primarily for **transparency, reproducibility, archival value, and licensing compliance**.

It may of course be forked and modified in accordance with the licenses inherited from upstream Free Pascal.

However, VandalFPC is **not intended to operate as a general community-maintained compiler fork**, and there is no expectation that external feature requests, pull requests, or general compiler changes will be accepted.

In particular:

* please do not use this repository as a general Free Pascal support forum;
* please do not raise issues here for unrelated upstream FPC bugs;
* please do not use this repository to report Vandal Engine bugs or request Vandal Engine support;
* pull requests may be declined even when technically valid if they do not serve the specific needs of the VandalSDK toolchain;
* upstream FPC issues should generally be raised with the Free Pascal project itself.

This repository should be understood as a **published, purpose-specific toolchain state**, not as an invitation to create a parallel general-purpose FPC development community.

Where useful, discussion or fixes may still be incorporated, but maintenance decisions will remain driven by the requirements of VandalSDK.

## Historical status

The complete history of the array-constructor behavior is still being investigated.

What is currently established is:

```text
FPC 3.2.2
    FAIL
    [0.1, 0.5, 3.0] reaches set handling and reports
    "Ordinal expression expected"

3.3.1 development, r42700 / 44bfa98a
    FAIL

3.3.1 development, r45199 / d83232f8
    FAIL

3.3.1 development, r46891 / 2ff18e48a0d
    PASS
```

Revision r46891 corresponds to the explicit addition of static-array conversion for array constructors.

There are older reports showing related set-versus-array resolution failures and repairs, suggesting that this area of the compiler has experienced repeated regressions or incomplete fixes over its history.

More historical versions may be tested in the future to establish when equivalent syntax previously worked and when it subsequently stopped working.

This section should be updated as that history becomes better understood.

## Building

Build instructions and bootstrap requirements will be documented once the VandalFPC build and artifact process has been finalized.

The intended model is that compiler releases from this repository will be built in a controlled environment and published as immutable artifacts.

Normal VandalSDK installations should not need to bootstrap Free Pascal from source.

## Licensing

VandalFPC retains the licenses of the upstream Free Pascal source from which it is derived.

The Free Pascal compiler sources are distributed under the GPL, while runtime components have their respective upstream licensing terms.

See the license files contained in this repository for the authoritative terms.

## Status

**Experimental / toolchain qualification in progress.**

The pinned compiler successfully provides the array-constructor and operator syntax required by Vandal Engine.

Target and backend qualification, particularly for x86-64 and AArch64, is ongoing before this compiler becomes the definitive VandalSDK toolchain.
