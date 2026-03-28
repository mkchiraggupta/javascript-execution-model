# Execution Context — Deep Dive

A single place to revise how JavaScript creates, runs, and tears down code. Use the **Quick Revision** section at the end for a 1-minute skim.

---

## 📘 Topics to Cover in execution-context/README.md

This document follows the structure below. For major ideas, each block uses:

- **Definition** — what it is in one sentence  
- **Explanation** — how it works  
- **Example** — minimal code when it helps  
- **Output** — what you see (if relevant)  
- **Why it happens** — engine / spec angle  

---

## 🧠 1. Introduction

### What is Execution Context?

- **Definition:** An execution context is the environment in which JavaScript code is **evaluated and executed** — it holds things like variables, `this`, and the outer scope link.
- **Explanation:** The engine does not run “bare” code in a void. For each chunk of running code (global script, function body, or `eval`), it builds a context: where variables live, how `this` is resolved, and how to look up names in outer scopes.
- **Example:** Loading a `.js` file creates the **global** execution context. Calling `foo()` creates a **function** execution context for `foo`.
- **Output:** N/A (conceptual).
- **Why it happens:** The ECMAScript spec defines execution through **execution contexts** so that variables, closures, and `this` behave consistently.

### Why it matters in JavaScript

- **Definition:** It is the backbone of **scope**, **hoisting**, **closures**, **`this`**, and the **call stack**.
- **Explanation:** Misunderstanding execution context leads to wrong mental models for async, React batching, and “why is my variable undefined here?”
- **Example:** Thinking “`let` is hoisted like `var`” without knowing **TDZ** causes surprise errors.
- **Output:** N/A.
- **Why it happens:** JS is **lexically scoped** but **dynamically** schedules work (event loop); execution context is the bridge between “where code was written” and “when it runs.”

### When it is created

- **Definition:** A new execution context is created when the engine **enters** global code, **calls** a function, or **enters** `eval` (and in modules, the module has its own top-level context behavior).
- **Explanation:**
  - **Global:** When the script starts (one GEC per realm / global object in classic scripts).
  - **Function:** Each **function invocation** gets its own FEC (not per function definition — per **call**).
  - **Eval:** Code run via `eval` can create its own context depending on strict/direct vs indirect eval (see section 2).
- **Example:**

```js
function a() {}
a(); // first call → one FEC
a(); // second call → another FEC
```

- **Output:** N/A.
- **Why it happens:** Each invocation needs its own **bindings** (parameters, locals) without clobbering other calls.

---

## 🔁 2. Types of Execution Context

### Global Execution Context (GEC)

- **Definition:** The context for code running at the **top level** of a script (not inside a function call).
- **Explanation:** There is typically **one active GEC** for that global environment. It sets up global bindings and, in browsers, associates with `window` (or `globalThis`).
- **Example:**

```js
var x = 1;
console.log(globalThis.x); // in browser: 1 (var on global object)
```

- **Output:** `1` (in non-module classic script with `var`).
- **Why it happens:** Top-level declarations need a **single** outer environment for the whole program.

### Function Execution Context (FEC)

- **Definition:** A context created for **one invocation** of a function.
- **Explanation:** Holds **arguments**, **local** `let`/`const`/`var`, **inner** functions’ lexical links, and **`this`** for that call.
- **Example:**

```js
function f(n) {
  return n * 2;
}
f(3);
```

- **Output:** `6`.
- **Why it happens:** Each call must isolate locals and parameters.

### Eval Execution Context (brief only)

- **Definition:** A context for code executed through `eval`.
- **Explanation:**
  - **Direct eval** (e.g. `eval('...')` in the same scope as the `eval` identifier) runs in the **calling** lexical environment (legacy behavior; still specified with constraints).
  - **Indirect eval** (e.g. `(0, eval)('...')`) typically runs in the **global** environment.
- **Example:**

```js
let x = 1;
function outer() {
  let x = 2;
  eval('console.log(x)'); // direct: sees outer’s x in many cases
}
```

- **Output:** Often `2` with direct eval (environment-dependent; avoid `eval` in real code).
- **Why it happens:** Spec treats eval specially for **backward compatibility** and security; **avoid** in production.

---

## ⚙️ 3. Phases of Execution Context (VERY IMPORTANT)

Every execution context (especially FEC) is understood in two phases: **Creation** (setup) and **Execution** (run line-by-line).

### 👉 Creation Phase

- **Definition:** Before any line of that context’s code runs, the engine **prepares** memory, bindings, hoisting behavior, `this`, and the **scope chain** / lexical links.
- **Explanation — Memory allocation:** Space is reserved for bindings (identifiers). Function declarations get a **full** binding early; `var` gets a binding initialized to `undefined`; `let`/`const` exist in the **temporal dead zone** until their initializer runs.
- **Variable handling:**

#### `var`

- **Definition:** Function-scoped (or global) binding, **hoisted** and initialized with `undefined` during creation (in its scope).
- **Explanation:** Accessing `var` before its line **does not throw** (you get `undefined`).
- **Example:**

```js
console.log(a);
var a = 5;
```

- **Output:** `undefined`
- **Why it happens:** Creation phase creates binding and sets `var` to `undefined` before execution assigns `5`.

#### `let` / `const` (TDZ)

- **Definition:** **Block-scoped** bindings; they exist after creation but are **uninitialized** until the declaration is **evaluated** — the gap is the **Temporal Dead Zone**.
- **Explanation:** Reading them before the declaration line throws `ReferenceError`.
- **Example:**

```js
console.log(b);
let b = 1;
```

- **Output:** `ReferenceError: Cannot access 'b' before initialization`
- **Why it happens:** Spec forbids use before initialization to catch bugs and keep semantics clear vs `var`.

#### Function declarations

- **Definition:** The **name** of a function declaration is fully available **throughout** its containing scope (after creation phase).
- **Explanation:** Body is associated with the binding during creation (hoisted “as a whole”).
- **Example:**

```js
foo();
function foo() { console.log('ok'); }
```

- **Output:** `ok`
- **Why it happens:** Declarations are instantiated in the environment record during creation.

#### `this` binding

- **Definition:** In creation (conceptually tied to context setup), **`this`** is determined by **how the function is called** (for normal functions), or lexically (for arrows).
- **Explanation:** Not “where it’s written” for regular functions — **call site** matters.
- **Example:**

```js
const obj = { m() { return this; } };
obj.m(); // this === obj
```

- **Output:** `true` when compared to `obj`.
- **Why it happens:** Spec’s `[[Call]]` sets `this` from the **reference** that triggered the call.

#### Scope chain creation

- **Definition:** Each context’s lexical environment has an **outer** reference — the chain used to **resolve** identifiers.
- **Explanation:** Inner sees outer; resolution walks **outward** until found or `ReferenceError`.
- **Example:** See section 5.
- **Output:** N/A.
- **Why it happens:** **Lexical nesting** of source code is captured in **environment records** and their `[[OuterEnv]]`.

### 👉 Execution Phase

- **Definition:** The engine runs code **line by line** (within that context), **assigning** values and **evaluating** expressions.
- **Explanation — Line-by-line execution:** After creation, assignments execute, functions **invoke** (pushing new contexts), and `let`/`const` leave TDZ when their declaration runs.
- **Value assignment:** `=` and compound assignments mutate bindings created earlier.
- **Example:**

```js
var x;
console.log(x);
x = 10;
console.log(x);
```

- **Output:**  
  `undefined`  
  `10`
- **Why it happens:** Creation prepared `x`; execution assigns when the assignment statement runs.

---

## 🧠 4. Call Stack & Execution Flow

### What is Call Stack?

- **Definition:** A **LIFO** structure of **active** execution contexts: the context on top is the one currently running.
- **Explanation:** `main`/global starts the stack; each **call** **pushes** a FEC; **return** **pops** it.

### LIFO concept

- **Definition:** **Last In, First Out** — the most recently entered function finishes first.
- **Example:**

```js
function first() { second(); console.log('first'); }
function second() { console.log('second'); }
first();
```

- **Output:**  
  `second`  
  `first`
- **Why it happens:** `second` is pushed on top of `first`; it runs to completion, pops, then `first` resumes.

### Push / Pop of execution contexts

- **Definition:** **Push** on **enter** (invoke), **pop** on **exit** (return or throw, after handling).
- **Explanation:** Async callbacks **do not** magically stay on the stack — they run later in a **new** stack turn when the callback is invoked.

### Stack overflow (brief)

- **Definition:** Too many nested synchronous calls exceed stack limits.
- **Example:**

```js
function r() { r(); }
r();
```

- **Output:** `RangeError: Maximum call stack size exceeded`
- **Why it happens:** Each call pushes a frame; infinite recursion never pops.

### Relation with execution context

- **Definition:** The stack **is** the ordered list of **active** execution contexts (conceptually).
- **Explanation:** Execution context = **what** runs; call stack = **which** context is active **now** and **who** called whom.

### 👉 Example: nested functions

```js
function a() {
  console.log('enter a');
  function b() {
    console.log('enter b');
    function c() {
      console.log('enter c');
    }
    c();
    console.log('leave b');
  }
  b();
  console.log('leave a');
}
a();
```

**Stack flow (conceptual):**

1. Push **GEC** (script).
2. Call `a` → push **FEC for `a`**.
3. Inside `a`, call `b` → push **FEC for `b`**.
4. Inside `b`, call `c` → push **FEC for `c`**.
5. `c` finishes → **pop** `c`.
6. `b` continues, finishes → **pop** `b`.
7. `a` continues, finishes → **pop** `a`.

**Output:**

```
enter a
enter b
enter c
leave b
leave a
```

**Why it happens:** Only the **top** frame runs; inner calls must complete (or yield in async) before outer resumes in sync code.

---

## 🔗 5. Scope Chain (VERY IMPORTANT)

### What is scope?

- **Definition:** **Where** a binding is **visible** and **legal** to access.
- **Explanation:** JS uses **lexical (static) scope**: scope is fixed by **source structure**, not by call stack alone.

### Lexical scope

- **Definition:** Inner functions “close over” outer bindings as defined in **nested blocks/functions** in source text.
- **Explanation:** The engine stores **outer environment** pointers — not “copy of values” for mutable bindings unless you capture them in a closure.

### How variable resolution works

- **Definition:** Look up the **current** environment record; if not found, follow **outer** reference; repeat until global or **not found** → `ReferenceError`.

### Scope lookup order

- **Explanation (typical):** **Current** lexical environment → **outer** → … → **global**. (With `with`, `eval`, or exotic objects, behavior can differ — avoid `with`.)

### 👉 Example: inner → outer → global

```js
const globalName = 'GLOBAL';

function outer() {
  const outerName = 'OUTER';

  function inner() {
    const innerName = 'INNER';
    console.log(innerName);  // found in inner
    console.log(outerName);  // not in inner → outer
    console.log(globalName); // not in inner/outer → global
  }

  inner();
}

outer();
```

- **Output:**  
  `INNER`  
  `OUTER`  
  `GLOBAL`
- **Why it happens:** Resolution walks the **scope chain** via outer lexical environments.

---

## 🧩 6. Lexical Environment

### What is Lexical Environment?

- **Definition:** A spec object pairing an **Environment Record** (stores bindings) with a reference to an **outer** Lexical Environment.
- **Explanation:** Execution contexts hold a **LexicalEnvironment** (and often a **VariableEnvironment** — for `var` in older models; in modern engines these often align per context type).

### Components

#### Environment Record

- **Definition:** Holds bindings for identifiers in that scope (`let`/`const`/`var`, functions, etc.).
- **Why it happens:** Needs a concrete place to resolve `x` when code runs.

#### Outer Reference

- **Definition:** Link to the **parent** lexical environment (the scope that **encloses** this one in source).
- **Why it happens:** Enables **closure** and **nested** visibility without searching the call stack.

### Relation with execution context

- **Definition:** An execution context **references** lexical environments used for **identifier resolution** and (where applicable) `this` binding in function code.
- **Explanation:** Think: **context** = “this run of a function/global”; **lexical env** = “the map of names + link outward.”

---

## 🔥 7. Hoisting

### What is hoisting?

- **Definition:** The **illusion** that declarations are moved to the top — actually, bindings are **created** during the **creation phase** before line-by-line execution.
- **Explanation:** Not all “hoisting” is the same: `var` vs `let` vs function declarations differ.

### Happens in creation phase

- **Definition:** Names are **registered** before statements in that scope execute.
- **Why it happens:** Spec’s **instantiation** step runs before execution for that scope.

### Difference: `var` / `let` / `const`

| Kind   | Created early? | Usable before line? | Initial value before line |
|--------|----------------|---------------------|---------------------------|
| `var`  | Yes            | Yes (as `undefined`) | `undefined`               |
| `let`  | Yes            | No (TDZ)            | uninitialized             |
| `const`| Yes            | No (TDZ)            | uninitialized             |

### Function hoisting

- **Function declaration:** Hoisted as a **full** callable binding.  
- **Function expression:** Only the **variable** (`var`/`let`) follows variable rules — the function value is assigned when that line runs.

```js
console.log(typeof fd); // 'function'
console.log(typeof fe); // 'undefined' (var) or TDZ error (let)

function fd() {}
var fe = function () {};
```

- **Why it happens:** Declaration vs expression — only declaration **instantiates** the function binding early.

---

## ⚠️ 8. Temporal Dead Zone (TDZ)

### What is TDZ?

- **Definition:** The period from **entering** the scope where `let`/`const` is declared until the **declaration statement** is **executed**, during which the binding **must not** be read.
- **Why it exists:** Prevents “temporal” bugs from using block-scoped variables before they are initialized; contrasts with sloppy `var` behavior.

### When it starts & ends

- **Starts:** When the scope is **entered** (block or function) — binding exists but is **uninitialized**.  
- **Ends:** Immediately **after** the initializer runs for `let`/`const`.

### Common errors

```js
if (true) {
  console.log(x); // TDZ
  let x = 1;
}
```

- **Output:** `ReferenceError`
- **Why it happens:** Reading `x` before `let x` executes violates TDZ rules.

---

## 🧠 9. `this` Binding

### Global context

- **Non-strict:** In browsers, top-level `this` is `globalThis`.  
- **Strict:** Still `globalThis` at true global script top level (environment-specific for modules).

### Function context

- **Definition:** For a normal function, **`this`** is set by the **call**:
  - `obj.method()` → `this` is `obj`
  - `fn()` (standalone) → `this` is `undefined` (strict) or `globalThis` (sloppy)

```js
'use strict';
function f() { return this; }
f(); // undefined
```

### Arrow functions

- **Definition:** Arrows **do not** have their own `this`; they close over **`this`** from the **enclosing** lexical scope.
- **Example:**

```js
const obj = {
  name: 'A',
  regular() {
    const arrow = () => this.name;
    return arrow();
  },
};
console.log(obj.regular()); // 'A'
```

### Strict vs non-strict mode

- **Explanation:** Sloppy mode “fixes” some `this` values to the global object; strict mode leaves `this` as `undefined` for plain calls — fewer footguns when refactoring.

---

## ⚠️ 10. Common Mistakes / Misconceptions

### Execution context vs scope

- **Misconception:** “Scope is the same as the call stack.”  
- **Reality:** **Scope** follows **lexical** outer links. **Stack** follows **call order**. A closure uses **outer env from creation**, not “who called me last.”

### Hoisting misunderstandings

- **Misconception:** “`let` is not hoisted.”  
- **Reality:** The binding is **created** before the line; it is in **TDZ**, not absent.

### TDZ confusion

- **Misconception:** “Temporal Dead Zone is a runtime pause.”  
- **Reality:** It is a **static rule** about **when reads are legal** relative to the declaration statement.

### `this` confusion

- **Misconception:** “`this` inside a function always refers to the object that owns the function.”  
- **Reality:** It depends on **call site** (or lexical `this` for arrows). Passing `obj.method` as a callback often **loses** `this` unless bound.

---

## 🎯 11. Interview Questions Section

### Concept-based

- What is an execution context vs a lexical environment?  
- What happens in the creation phase vs execution phase?  
- How does the scope chain differ from the call stack?  
- Why does `var` log `undefined` but `let` throws before declaration?

### Scenario-based

- Explain what `this` is in `setTimeout(obj.m, 0)` vs `setTimeout(() => obj.m(), 0)`.  
- Why can an inner function still read outer variables after outer returns? (closures + outer env)  
- What changes in strict mode for `this` in a free function call?

### Output-based

- Predict output for nested `var`/`let`, function declaration vs expression, and arrow vs regular method as callback.

---

## 🧪 12. Real Code Walkthrough (IMPORTANT)

### Full example

```js
var a = 1;

function outer() {
  var b = 2;

  function inner() {
    console.log(a, b);
    var c = 3;
    console.log(c);
  }

  inner();
}

outer();
console.log(typeof c);
```

### Creation phase (high level)

- **GEC:** `var a` binding created → `undefined`. `function outer` binding created → points to function object.
- **When `outer()` is invoked — FEC `outer`:** `var b` → `undefined`; `function inner` fully instantiated.
- **When `inner()` is invoked — FEC `inner`:** `var c` → `undefined` (hoisted inside `inner`).

### Execution phase (high level)

- GEC runs: `a = 1`.
- `outer()` runs: `b = 2`; calls `inner()`.
- `inner()` runs: first `console.log(a, b)` → resolves `a` via outer chain to global `1`, `b` via outer to `outer`’s `2`. Then `c = 3`; logs `3`.
- After `inner` returns, back in `outer`, then pop. Final line: `c` is **not** in global scope → `typeof c` is `'undefined'` (no global `c` binding).

### Stack flow

`GEC` → push `outer` → push `inner` → pop `inner` → pop `outer` → `GEC` continues.

### Scope resolution

- `a` in `inner`: not in `inner` → outer `outer`? no `a` → global → `1`.  
- `b` in `inner`: not in `inner` → `outer` → `2`.  
- `c` in `inner`: local `var c` (hoisted, then assigned).

### Output

```
1 2
3
undefined
```

### Why it happens

- **Hoisting** makes `var c` exist before its line but uninitialized until assignment line for the **log** — first log only uses `a`, `b`; second log runs after `c = 3`.  
- **`c`** is **function-scoped** to `inner`, not visible at global `typeof c`.

---

## 🧾 13. Quick Revision Section (1-Minute Recap)

- **Execution context** = environment for running a chunk of code (global / function / eval).  
- **Creation phase** = bindings, hoisting rules, outer links, `this` rules prepared.  
- **Execution phase** = run code, assign values, call functions (new contexts).  
- **Call stack** = LIFO chain of **active** contexts; recursion can overflow it.  
- **Scope chain** = lexical outer links for **name lookup** (not the same as stack).  
- **Lexical environment** = environment record + outer reference.  
- **`var`** hoists to `undefined`; **`let`/`const`** hoist to TDZ until the line.  
- **Function declarations** hoisted whole; **expressions** follow their variable.  
- **`this`:** regular functions → call site; arrows → lexical `this`.  
- **Closures** = function + captured outer environments.

---

## 🚀 14. Real-World Relevance

- **Debugging:** “Wrong value” often = wrong **closure**, wrong **`this`**, or TDZ/hoisting surprise.  
- **React:** Event handlers, class vs hooks, and stale closures tie to **lexical env** + **when** callbacks run (async), not only execution context stack.  
- **Closures:** Modules, memoization, and callbacks rely on **outer** environment retention.  
- **Async foundation:** Synchronous stack **pops** before timers run; callbacks start a **new** stack later — execution context idea still applies **per turn**.

---

## 🔥 Pro Tip (Important)

While writing or revising in Cursor, for **each** topic use this mini-template:

1. **Definition** — one tight sentence  
2. **Explanation** — steps or rules  
3. **Example** — smallest code that shows it  
4. **Output** — if it’s observable  
5. **Why it happens** — spec/engine intuition  

This keeps notes **interview-ready** and **fast to skim** later.
