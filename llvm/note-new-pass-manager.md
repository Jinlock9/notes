#### What pass managers do:

They schedule optimizations and analysis passes on code (IR) in a specific order and manage caching of analysis results to avoid expensive re-computations.

---
# LLVM's New Pass Manager

LLVM's **New Pass Manager** (New PM) is ==the current, default infrastructure for the optimization pipeline (middle-end)== and is actively replacing the legacy pass manager. It offers a more modern design, better analysis management, and support for parallel execution, leading to performance gains in large codebases like Chrome.

---
## Key Features 

### 1. Explicit and Hierarchical Pass Nesting

Rather than relying on implicit scheduling logic, the New Pass Manager uses explicit pass management at each IR level.

**Key feature:**  
The pipeline structure is explicitly defined. For example, a `ModulePassManager` contains a `FunctionPassManager`, which may in turn contain a `LoopPassManager`.

**Example:**

```cpp
ModulePassManager MPM;
FunctionPassManager FPM;
FPM.addPass(InstCombinePass());
MPM.addPass(createModuleToFunctionPassAdaptor(std::move(FPM)));
```

Here, `InstCombinePass` is guaranteed to run on every function, and its position in the pipeline is explicit.

**Benefit:**  
The execution order of passes is easy to understand and reason about, which simplifies debugging and pipeline customization.

### 2. Concept-Based Polymorphism

The New PM departs from the inheritance-heavy design of the legacy pass manager.

**Key feature:**  
Passes are implemented using C++ templates and CRTP-based mix-ins (`PassInfoMixin<T>`) instead of inheriting from a single virtual base class.

**Example:**

```cpp
struct MyPass : PassInfoMixin<MyPass> {
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM) {
    // transform F
    return PreservedAnalyses::all();
  }
};
```

This pass does not inherit from a virtual `Pass` base class and does not require registration macros.

**Benefit:**  
This reduces boilerplate, improves type safety, and avoids the complexity of deep inheritance hierarchies.

### 3. Separate, Lazy Analysis Management

Analysis computation is fully decoupled from IR transformation passes.

**Key feature:**  
==A dedicated `AnalysisManager` computes analyses lazily—only when they are requested—and caches the results==.

**Example:**

```cpp
auto &DT = AM.getResult<DominatorTreeAnalysis>(F);
```

The Dominator Tree is computed only if this line is executed and reused if another pass requests it later.

**Benefit:**  
Analyses that are never used by subsequent passes are not computed, avoiding unnecessary work.

### 4. Precise Analysis Invalidation

The New PM enables fine-grained control over analysis validity after IR changes.

**Key feature:**  
Each transformation pass returns a `PreservedAnalyses` object that ==explicitly states which analyses remain valid==.

**Example:**

```cpp
PreservedAnalyses PA;
PA.preserve<DominatorTreeAnalysis>();
return PA;
```

This indicates that only the Dominator Tree remains valid, while other analyses are invalidated.

**Benefit:**  
This minimizes redundant recomputation of expensive analyses, significantly improving compilation performance.

### 5. Pass Pipelining and Improved Locality

The execution model is designed to improve memory access efficiency.

**Key feature:**  
==All relevant passes are run on a single IR unit (such as one function) before moving on to the next unit==.

**Example:**

```text
Function A: Pass1 → Pass2 → Pass3
Function B: Pass1 → Pass2 → Pass3
```

Instead of running `Pass1` on all functions first, then `Pass2`, the New PM completes all passes on one function at a time.

**Benefit:**  
Keeping the active IR unit in the CPU cache improves locality, reduces memory traffic, and speeds up compilation.

### 6. Design for Concurrency

The architecture is designed with multi-core execution in mind.

**Key feature:**  
==Independent passes—such as function passes operating on different functions—can be executed safely in parallel.==

**Example:**

```text
Thread 1: Optimize Function foo()
Thread 2: Optimize Function bar()
Thread 3: Optimize Function baz()
```

Each thread operates on a separate function with its own analysis state, avoiding data races.

**Benefit:**  
The compiler can utilize available CPU cores more effectively, reducing overall build times for large codebases.

---
## Expected Benefits

#### Performance Benefits (Compile Time)

- ==**Faster Compilation==:** By reducing redundant analysis computations and improving CPU cache locality through pipelining, the New PM speeds up the overall compilation process, with notable gains in large codebases.

- **Scalability for Multi-core Systems:** The design inherently supports safe parallel execution of passes across different functions, utilizing modern hardware effectively.

#### Project & Code Health Benefits

- **Improved Maintainability:** The explicit and modular design of the New PM makes it easier for developers to reason about optimization pipelines, debug issues, and add new passes without introducing subtle bugs.

- **Robustness:** The clear separation of concerns (passes from analyses) and predictable analysis management leads to a more stable compiler with fewer non-deterministic crashes or miscompilations.

---
## Current State of New Pass Manager

The current state of the LLVM New Pass Manager (New PM) is characterized by full adoption in the middle-end and a transitional hybrid approach in the backend pipeline:

#### Middle-End Pipeline (LLVM IR Optimizations)

- **Status:** Fully adopted and is the default infrastructure.

- **Implementation:** The entire optimization pipeline uses the New PM's explicit, hierarchical structure (Module -> CGSCC -> Function -> Loop managers), efficient analysis caching, and support for parallelism. The legacy pass manager is no longer used for middle-end operations. 

#### Backend Pipeline (Code Generation, Machine IR)

- **Status:** In a transitional phase; relies on a hybrid approach.

- **Implementation:**
    - The primary execution infrastructure for Machine IR (MIR) passes remains the **legacy pass manager**.
    - Many individual generic code-gen passes (e.g., `MachineCSE`, `TwoAddressInstructionPass`) have been **refactored** to use the modern New PM API style (`PassInfoMixin<T>`).
    - These refactored passes run within the legacy system using **adapter layers**.

- **In short**: 

	- It is currently possible to implement Machine IR (MIR) passes using the **New Pass Manager (New PM) API** (specifically `PassInfoMixin<T>`) and register them within the **legacy pass manager** infrastructure. However, the comprehensive refactoring of the entire codegen execution pipeline into a native New PM style remains under development.

	- Consequently, ==the codegen pipeline does not yet fully benefit from the New PM's advanced analysis management capabilities==, such as precise analysis invalidation, pass pipelining, or inherent concurrency.

---
## CGSCC Benefits

#### What is CGSCC?

A **CGSCC (Call Graph Strongly Connected Component)** is a group of functions in the call graph that can all reach each other through function calls. In practice, this means recursive functions or call cycles. A **CGSCC pass** runs on these groups in a **bottom-up order**, so functions that are called are handled before the functions that call them.
#### Why is CGSCC Needed? (Especially for the Inliner)

Inlining is not a single-function optimization. To decide whether a call should be inlined, the compiler often needs to:

- Look at properties of the callee (size, attributes, profile data).
- Look through simple wrapper functions to see what they actually call.
- Understand short call chains, not just one call edge.

CGSCC provides a natural unit for this. It lets the inliner reason about related functions together and guarantees that callees are analyzed before callers. Without CGSCC, inlining decisions become either too conservative or too expensive to compute.
#### Problems with the Legacy Pass Manager

The legacy pass manager made this difficult:

- It did not keep a persistent call graph.
- SCCs were handled as temporary lists of functions.    
- There was no stable way to cache analysis results for an SCC.
- Accessing analysis results for functions outside the current SCC was inefficient.
- Once a function was processed, it was never revisited, even if the call graph later changed.

As a result, the inliner often lacked enough information to make good decisions.

#### New Pass Manager: What Changed and Why It Helps

The New Pass Manager fixes these issues in a practical way:

- **Persistent Call Graph**  
    The full call graph is kept in memory, allowing analyses to be cached and reused.
    
- **Incremental Updates**  
    When optimizations change calls between functions, the call graph is updated instead of rebuilt.
    
- **SCC Splitting and Revisiting**  
    If a call cycle is broken, the New PM detects this, splits the SCC, and reprocesses the affected functions in the correct order.
    
- **Better Analysis Access for the Inliner**  
    The inliner can efficiently inspect analysis and profile data for callees, even beyond the current SCC.
    

**Practical benefit:**  
The inliner can make better decisions, especially for wrapper functions and profile-guided optimization. This directly leads to better generated code and measurable performance improvements.

---

## Update
- `2025.12.15`: New Pass Manager, Key Features, Expected Benefits, Current State of New Pass Manager, and CGSCC Benefits.