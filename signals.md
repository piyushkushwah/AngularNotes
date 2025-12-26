% ⚡ Angular Signals — Complete Interview Notes (2025)
% Senior Frontend Developer Preparation Guide
% December 2025

---

# ⚡ Angular Signals — Complete Interview Notes

## Senior Frontend Developer Preparation Guide (2025)

---

\newpage

## 📋 Table of Contents

1. Why Signals Were Introduced
2. Core Reactive APIs (5 Essential Functions)
3. Reactive Context Explained
4. Signal Lifecycle (Push → Pull)
5. toSignal() — RxJS Interoperability
6. assertNotInReactiveContext()
7. Effect Execution Timing
8. Interview Q&A (20+ Questions with Answers)
9. Common Patterns & Use Cases
10. Do's & Don'ts (Golden Rules)
11. Performance & Best Practices

---

\newpage

## 1. Why Signals Were Introduced (The Problem Signals Solve)

### **The Zone.js Problem**

**Before Signals:**
- Angular used **Zone.js** to monkey-patch all async events
- When ANY event happened (click, timer, HTTP call), Angular checked the **entire component tree**
- Even if only 1 component changed, 99% of checks were wasted CPU cycles
- Performance cost: massive for large applications

**Timeline Example:**
```
User clicks button in Header
    ↓
Zone.js: "Something changed somewhere"
    ↓
Angular checks EVERY component on page:
- AppComponent ✗
- Header ✓ (1% actual change)
- Sidebar ✗
- UserProfile ✗
- Footer ✗
    ↓
Repeat on EVERY async event = terrible performance
```

### **Signals Solution**

- **Fine-grained reactivity:** Only updates components that depend on changed signal
- **Zone-less path:** Eliminates Zone.js overhead (20KB+ reduction)
- **Direct dependency graph:** Angular knows EXACTLY what depends on what
- **Synchronous updates:** No async complexity

**Result:** Only affected 5% of UI updates, leaving 95% alone = **massive perf gain**

---

\newpage

## 2. Core Reactive APIs (5 Essential Functions)

### **Table of All 5**

| Function | Type | Purpose | When to Use | Writable? |
|:---------|:-----|:--------|:-----------|:----------|
| **signal()** | Writable | Holds mutable reactive state | Local component state | ✅ Yes |
| **computed()** | Read-only | Derives values from signals (cached) | Derived data, caching | ❌ No |
| **effect()** | Side Effect | Runs code when signals change | Logging, DOM, storage sync | ❌ No |
| **linkedSignal()** | Read+Write | Editable copy synced to source | Edit forms, drafts | ✅ Yes |
| **toSignal()** | From Observable | Converts RxJS to signal | HTTP, NgRx, Observables | ❌ No |

---

### **2.1 signal() — Writable Signal**

```typescript
const count = signal(0);              // Create with initial value
count.set(5);                         // Replace entire value
count.update(prev => prev + 1);      // Update based on previous
console.log(count());                // Read current value (function call!)

// Type annotation (optional, auto-inferred)
const username: WritableSignal<string> = signal('John');
```

**Key Points:**
- ✅ Synchronous — value available immediately
- ✅ Writable — has `.set()` and `.update()` methods
- ✅ Readable — call `signal()` to get current value
- ✅ Tracks dependents automatically

---

### **2.2 computed() — Read-Only Derived Signal**

```typescript
const count = signal(1);
const double = computed(() => count() * 2);  // Reads count
const triple = computed(() => count() * 3);

console.log(double());  // 2 (reads count inside, so tracked)
```

**Key Points:**
- ✅ Caches result — doesn't recalculate until dependency changes
- ✅ Read-only — cannot `.set()` or `.update()`
- ✅ Automatic tracking — records which signals it reads
- ✅ Lazy — doesn't run until someone reads it
- ❌ Cannot write to signals inside it

---

### **2.3 effect() — Async Side Effect Runner**

```typescript
const count = signal(0);

effect(() => {
  console.log('Count changed to:', count());
  localStorage.setItem('count', count().toString());
});
```

**Key Points:**
- ✅ Runs at least once immediately
- ✅ Re-runs when dependency signals change
- ✅ Async execution during change detection
- ✅ Supports cleanup with `onCleanup()` callback
- ✅ Auto-cleanup on component destroy
- ❌ Don't use for state propagation (use `computed()` instead)

**Common Use Cases:**
- Logging and debugging
- Syncing to localStorage/sessionStorage
- Third-party library updates (charts, maps)
- Custom DOM manipulation

---

### **2.4 linkedSignal() — Editable Derived Signal**

```typescript
const user = signal({id: 1, name: 'John'});

const editingUser = linkedSignal(
  () => user(),  // Source
  { equal: (a, b) => a.id === b.id }  // Custom equality
);

editingUser.set({id: 1, name: 'Jane'});  // Can edit!
console.log(editingUser());  // Shows Jane

user.set({id: 2, name: 'Bob'});  // Different user!
console.log(editingUser());  // Resets to Bob (different ID)

user.set({id: 1, name: 'Johnny'});  // Same ID!
console.log(editingUser());  // KEEPS Jane edits (same ID, ignores reset)
```

**Key Points:**
- ✅ Starts as copy of source signal
- ✅ You can edit it with `.set()` / `.update()`
- ✅ Keeps your edits when source changes to SAME entity (same ID)
- ✅ Resets when source changes to DIFFERENT entity
- ✅ Perfect for edit forms

---

### **2.5 toSignal() — Observable to Signal Bridge**

```typescript
import { toSignal } from '@angular/core';

@Component({ ... })
export class UserProfile {
  readonly user = toSignal(
    this.userService.getUser$(),
    { initialValue: null }  // REQUIRED!
  );

  // Now use like any other signal
  userName = computed(() => this.user()?.name ?? 'Loading...');

  // In template: {{ user()?.email }}
}
```

**Key Points:**
- ✅ Converts Observable → Signal
- ✅ Requires `{initialValue: ...}` (handles async timing)
- ✅ Auto-subscribes on creation
- ✅ Auto-unsubscribes on destroy
- ✅ Read-only signal (no `.set()`)
- ❌ **Cannot use inside `computed()` or `effect()`** (throws error)
- ✅ Use `assertNotInReactiveContext()` for protection

**When to Use:**
- HTTP calls
- NgRx store selectors
- RxJS pipelines
- Third-party Observables

---

\newpage

## 3. Reactive Context — The Tracking Phase

### **Definition**

> **Reactive Context** = The period when Angular **records which signals are being read** to establish dependencies.

Think of it as: **Angular pressing the RECORD button** to note which signals are used.

### **When Angular Enters Reactive Context**

Angular enters whenever it needs to track dependencies:

```typescript
// ✅ INSIDE Reactive Context:
computed(() => count())        // Recording ON - count is tracked

effect(() => console.log(count()))  // Recording ON - count is tracked

{{ count() }}                    // Template reads - Recording ON

// ❌ OUTSIDE Reactive Context:
function someFunction() {
  count();                       // Recording OFF - ignored!
}

setTimeout(() => count(), 100)  // Recording OFF
```

### **Visual Diagram**

```
┌─────────────────────────────────────────┐
│ REACTIVE CONTEXT = ON (Recording)       │
├─────────────────────────────────────────┤
│                                         │
│  computed(() => {                       │
│    const x = count();    ← TRACKED!     │
│    return x * 2;                        │
│  });                                    │
│                                         │
│  Angular notes: "computed depends on count"
│                                         │
└─────────────────────────────────────────┘


┌─────────────────────────────────────────┐
│ REACTIVE CONTEXT = OFF (Not Recording)  │
├─────────────────────────────────────────┤
│                                         │
│  function test() {                      │
│    count();              ← IGNORED!     │
│  }                                      │
│                                         │
│  Angular doesn't note this read         │
│                                         │
└─────────────────────────────────────────┘
```

### **Why It Matters**

Without tracking, Angular wouldn't know:
- Which signals changed
- What depends on what
- What to update

With tracking, Angular builds a **producer → consumer graph**:

```
count ────┐
          ├──→ double (computed)
          ├──→ effect()
          └──→ template
```

When `count.set(5)` → Angular KNOWS to update only those 3 things!

---

\newpage

## 4. Signal Lifecycle — Push → Pull Model

### **Two Phases**

```
Phase 1: PUSH (Notify)
  count.set(5)
    ↓
  Marks ALL consumers "dirty" instantly
  (No recomputation yet!)

Phase 2: PULL (Recompute)
  {{ double() }}  ← Template reads
    ↓
  Sees "dirty" flag
    ↓
  Recomputes double → 10
    ↓
  Updates DOM
```

### **Complete Timeline**

```
Time ──────────────────────────────────→

1. count.set(5)
   ↓ PUSH PHASE (synchronous)
   [count] marked dirty
   [double] marked dirty
   [effect] marked dirty
   Template marked dirty
   (All happen instantly!)

2. {{ double() }} runs in change detection
   ↓ PULL PHASE (synchronous)
   Double sees "dirty" → recomputes
   double = 10
   Caches result
   Template updates DOM

3. effect() runs in change detection
   ↓ effect sees count() = 5
   Logs "New count: 5"
   Syncs to localStorage
```

### **Key Characteristics**

| Aspect | Details |
|:-------|:--------|
| **Synchronous** | No waiting — values available immediately |
| **Fine-grained** | Only dependents update, rest untouched |
| **Cached** | `computed()` caches result until dependency changes |
| **Deterministic** | Same input always = same output |
| **Trackable** | Can trace exact dependency graph |

### **Comparison: Before vs After**

```
BEFORE (Zone.js):
Any event → Check everything → Update everything
           ✗ Wasted CPU

AFTER (Signals):
count.set() → Mark dirty [double, effect] → Update only those
            ✓ Precise updates
```

---

\newpage

## 5. toSignal() — RxJS Interoperability

### **Why toSignal() Exists**

Modern Angular uses signals for local state, but:
- Many APIs return Observables (HTTP, NgRx, etc.)
- You need to bridge Observable → Signal
- `toSignal()` does this conversion automatically

### **How It Works**

```typescript
import { toSignal } from '@angular/core';
import { interval } from 'rxjs';

// Observable: emits values over time
const interval$ = interval(1000);

// Convert to Signal with initial value
const intervalSignal = toSignal(interval$, {
  initialValue: 0  // REQUIRED! Handles async timing
});

// Now use like any signal
console.log(intervalSignal());  // 0 (initial)
// After 1s → 1
// After 2s → 2
```

### **Complete HTTP Example**

```typescript
@Component({
  selector: 'app-user',
  template: `
    <div *ngIf="user(); else loading">
      <h1>{{ user().name }}</h1>
      <p>{{ user().email }}</p>
    </div>
    <ng-template #loading>Loading...</ng-template>
  `
})
export class UserComponent {
  // Convert HTTP Observable to Signal
  readonly user = toSignal(
    this.userService.getUser(),
    { initialValue: null }
  );

  constructor(private userService: UserService) {}
}
```

### **Key Points**

| Point | Details |
|:------|:--------|
| **Required** | `{initialValue: ...}` — handles timing before Observable emits |
| **Auto-subscribe** | Subscribes immediately when created |
| **Auto-unsubscribe** | Unsubscribes when component/service destroyed |
| **Type** | Read-only signal (no `.set()`) |
| **Lifecycle** | Tied to injection context (constructor, ngOnInit) |

### **❌ CRITICAL ERROR: Can't Use Inside Reactive Context**

```typescript
// ❌ This throws error:
effect(() => {
  const user = toSignal(getUserObservable$());
  console.log(user());
});

// Why? Angular thinks subscription is a dependency → infinite loops

// ✅ CORRECT - Create outside:
readonly user = toSignal(getUserObservable$(), {
  initialValue: null
});

effect(() => {
  console.log(this.user());  // Safe, just reads
});
```

### **Compare: Observable vs Signal**

```typescript
// Observable (async, cold)
userService.getUser$().subscribe(user => {
  console.log(user);  // Gets called when data arrives
});

// Signal (sync, always has value)
const user = toSignal(userService.getUser$(), {initialValue: null});
console.log(user());  // Returns current value immediately
```

---

\newpage

## 6. assertNotInReactiveContext()

### **Purpose**

Prevents certain functions from running inside reactive contexts (`computed()`, `effect()`).

**Used by:** Angular internals, signal-creation utilities, data-fetching APIs.

### **Why It Exists**

Some operations (creating signals, subscribing, side effects) would cause **infinite loops** or **corrupt state** if run during dependency tracking.

```typescript
// Example: What it prevents

const count = signal(0);

// ❌ BAD: Creating signal inside effect
effect(() => {
  const derived = toSignal(someObservable$);  // ❌ Error!
  // Angular: "Creating signal during tracking?"
  // Result: Infinite loops
});
```

### **How It Works**

```typescript
function toSignal(observable$) {
  // Check: Am I being called inside reactive context?
  assertNotInReactiveContext(toSignal);
  
  // If yes → Error thrown immediately
  // If no → Continue safely
  
  return new SignalFromObservable(observable$);
}
```

### **Real Usage Example**

```typescript
import { assertNotInReactiveContext, toSignal } from '@angular/core';

@Component({...})
export class UserList {
  // ✅ SAFE: Called outside reactive context (in constructor)
  readonly users = toSignal(
    this.userService.getUsers$(),
    { initialValue: [] }
  );

  constructor(private userService: UserService) {}

  loadMoreUsers() {
    // ✅ SAFE: Called from event handler (not reactive context)
    this.users = toSignal(
      this.userService.getMoreUsers$(),
      { initialValue: this.users() }
    );
  }

  ngOnInit() {
    // ❌ WRONG: Called inside ngOnInit with effect
    effect(() => {
      const moreUsers = toSignal(this.userService.getMoreUsers$());
      // Throws: "Cannot create signal in reactive context"
    });
  }
}
```

### **Rule of Thumb**

```
Safe to create signals/toSignal():
✅ Constructor
✅ Regular functions
✅ Event handlers
✅ Service methods

UNSAFE - throws error:
❌ Inside computed()
❌ Inside effect()
❌ Inside template {{ }} with tracking
❌ During dependency recording
```

---

\newpage

## 7. Effect Execution Timing

### **Two Types of Effects**

Angular runs effects at different times based on WHERE they're created.

#### **Type 1: View Effect**

```typescript
@Component({
  selector: 'app-counter',
  template: `<button (click)="increment()">Count: {{ count() }}</button>`
})
export class CounterComponent {
  count = signal(0);

  constructor() {
    // ✅ This is a VIEW EFFECT
    effect(() => {
      console.log('Count changed:', this.count());
    });
  }

  increment() {
    this.count.update(c => c + 1);
  }
}
```

**Timing:**
```
Change Detection Cycle:
  ↓
View Effect runs ← BEFORE component template is checked
  ↓
Component template updates
  ↓
DOM renders
```

#### **Type 2: Root Effect**

```typescript
@Injectable({ providedIn: 'root' })
export class AppService {
  count = signal(0);

  constructor() {
    // ✅ This is a ROOT EFFECT
    effect(() => {
      console.log('Root effect:', this.count());
      localStorage.setItem('count', this.count().toString());
    });
  }
}
```

**Timing:**
```
Change Detection Cycle:
  ↓
Root Effect runs ← BEFORE ALL components are checked
  ↓
All View Effects run
  ↓
All components update
  ↓
DOM renders
```

### **Visual Timeline**

```
              START CHANGE DETECTION
                        ↓
              ┌─────────────────────┐
              │  Root Effects Run   │ (AppService effect)
              └─────────────────────┘
                        ↓
┌─ Component 1 ──────────────────────┐
│  ┌─────────────────────┐            │
│  │ View Effects Run    │ (this effect)
│  └─────────────────────┘            │
│  Check Component → Update Template   │
└──────────────────────────────────────┘
                        ↓
┌─ Component 2 ──────────────────────┐
│  ┌─────────────────────┐            │
│  │ View Effects Run    │            │
│  └─────────────────────┘            │
│  Check Component → Update Template   │
└──────────────────────────────────────┘
                        ↓
              ┌─────────────────────┐
              │   Render DOM        │
              └─────────────────────┘
```

### **Both Effects Share Common Traits**

| Trait | Details |
|:-----|:--------|
| **Async** | Run asynchronously during change detection |
| **Re-run** | Re-run if dependency changed during execution |
| **Cleanup** | Auto-cleanup when destroyed |
| **onCleanup()** | Can register cleanup callback |

### **Effect Cleanup Example**

```typescript
effect((onCleanup) => {
  const timer = setTimeout(() => {
    console.log('Timer fired');
  }, 1000);

  // Register cleanup
  onCleanup(() => {
    clearTimeout(timer);  // Runs before next effect or destruction
  });
});
```

---

\newpage

## 8. Interview Q&A — 20+ Questions with Answers

### **Q1: What are signals and why were they introduced?**

**Answer:**
> Signals are reactive state primitives that enable **fine-grained reactivity** without relying on Zone.js.
>
> **Why introduced:** Angular used Zone.js to monkey-patch async events, which caused Angular to check the **entire component tree** on every event — wasting 95% of CPU cycles. Signals introduce a **direct producer → consumer graph**, so only affected components update. This path enables **Zone-less change detection** and massive performance gains.

---

### **Q2: Difference between signal(), computed(), and effect()?**

**Answer:**
- **signal():** Holds mutable state. Has `.set()` and `.update()` methods. Writable.
- **computed():** Derives values from signals. Read-only. Auto-caches. Only recalculates when dependency changes.
- **effect():** Runs code when signals change. Async side effects. Used for logging, DOM, storage sync.

---

### **Q3: Are signals synchronous or asynchronous?**

**Answer:**
> **Signals are synchronous.** When you call `.set()` or `.update()`, the value changes immediately and is available on the next line. No waiting like Observables or Promises.
>
> Exception: `effect()` itself runs **asynchronously** (during change detection), but the signal updates within it are still synchronous.

---

### **Q4: What is a reactive context?**

**Answer:**
> A **reactive context** is the period when Angular is **tracking which signals are being read** to build a dependency graph.
>
> Analogy: Angular is "recording mode ON" to note all signal reads. Later, when those signals change, Angular knows exactly what depends on them.
>
> Enters during: `computed()`, `effect()`, template signal reads.

---

### **Q5: Why does toSignal() throw an error inside effect()?**

**Answer:**
> `toSignal()` creates subscriptions to Observables. If called inside `effect()` (which is a reactive context), Angular tracks that subscription **as a dependency**.
>
> When the Observable emits (dependency "changes"), it re-triggers the effect → which creates new subscriptions → infinite loop → stack overflow.
>
> **Solution:** Create `toSignal()` outside the reactive context (constructor, regular functions).

---

### **Q6: What is assertNotInReactiveContext()?**

**Answer:**
> A **guard function** that prevents code from running inside reactive contexts. Used by `toSignal()` and other signal-creation APIs.
>
> If you try to call `toSignal()` inside an `effect()`, it throws: "Cannot create signal in reactive context."
>
> **Why:** Protects against infinite loops, context pollution, and state corruption.

---

### **Q7: What happens if you call set() inside a computed()?**

**Answer:**
> **Infinite loop → stack overflow.**
>
> `computed()` runs and calls `set()`, which marks the computed "dirty", so it runs again, calls `set()` again... forever.
>
> **Rule:** `computed()` must be pure (read-only).

---

### **Q8: What is the signal lifecycle (Push → Pull)?**

**Answer:**
> **Push Phase:** `count.set(5)` marks all dependent computeds/effects "dirty" instantly (no recomputation).
>
> **Pull Phase:** Later, when template reads `{{ double() }}`, it sees "dirty" → recomputes → returns 10 → updates DOM.
>
> Both happen **synchronously** in the same JS tick.

---

### **Q9: When does effect() run?**

**Answer:**
> **At least once immediately** (on creation), then **whenever a dependency signal changes**.
>
> It runs **asynchronously during change detection**, not immediately on signal change.

---

### **Q10: Difference between View Effect and Root Effect?**

**Answer:**
> **View Effect:** Created inside component/local service. Runs BEFORE that component's change detection.
>
> **Root Effect:** Created in root-provided service. Runs BEFORE ALL components' change detection.
>
> Both are async during CD cycle, but Root runs first.

---

### **Q11: When should you use linkedSignal()?**

**Answer:**
> Use `linkedSignal()` when you need an **editable copy** of a derived signal that:
> - Starts as copy of source
> - You can edit with `.set()` / `.update()`
> - Stays in sync with source BUT preserves your edits if source changes to the same entity (e.g., same ID)
> - Resets if source changes to a different entity
>
> **Use case:** Edit forms (edit user profile, keep draft even if parent user updates).

---

### **Q12: How do you create a writable vs read-only signal?**

**Answer:**
> **Writable:** `const count = signal(0);` (has `.set()` and `.update()`)
>
> **Read-only:** `const double = computed(() => count() * 2);` (no `.set()`)
>
> Also: `toSignal()` creates read-only signals.

---

### **Q13: What is untracked() used for?**

**Answer:**
> Allows you to **read a signal without tracking it as a dependency** inside an `effect()`.
>
> **Example:**
> ```typescript
> effect(() => {
>   const userId = userId();  // tracked
>   const time = untracked(() => new Date());  // NOT tracked
>   logEvent({userId, time});
> });
> ```
>
> When `time` signal changes, the effect **won't** re-run (because untracked).

---

### **Q14: Can you modify objects inside a signal without calling set()?**

**Answer:**
> ❌ **No.** Signals are shallow-checked.
>
> ```typescript
> const user = signal({name: 'John'});
> user().name = 'Jane';  // ❌ Won't trigger update!
>
> user.set({...user(), name: 'Jane'});  // ✅ Correct
> ```

---

### **Q15: How do you handle async data loading with signals?**

**Answer:**
> Use `toSignal()` with HTTP Observable:
>
> ```typescript
> readonly user = toSignal(
>   this.userService.getUser$(),
>   { initialValue: null }  // Required!
> );
> ```
>
> The `initialValue` handles the timing before Observable emits.

---

### **Q16: What's the advantage of signals over RxJS subjects?**

**Answer:**
> - **Simpler:** No `.subscribe()` boilerplate
> - **Synchronous:** Always have a value (no null checks)
> - **Type-safe:** Full TypeScript support
> - **Performant:** Fine-grained reactivity, no Zone.js
> - **Fewer operators:** Computed replaces map, filter, etc.

---

### **Q17: Can you use signals in an NgRx store?**

**Answer:**
> Angular signals are for **local component state**. NgRx is for **global app state**.
>
> You can bridge with: `toSignal(store.select(selectUser))` to convert NgRx Observable → Signal for local use.
>
> Don't replace NgRx with pure signals for large apps.

---

### **Q18: How do effects handle cleanup?**

**Answer:**
> Use the `onCleanup()` callback:
>
> ```typescript
> effect((onCleanup) => {
>   const timer = setTimeout(() => { /* */ }, 1000);
>
>   onCleanup(() => {
>     clearTimeout(timer);  // Runs before next effect or destroy
>   });
> });
> ```
>
> Also auto-cleanup on component/service destroy.

---

### **Q19: What's the difference between zone-less and Zone.js?**

**Answer:**
> **Zone.js:** Monkey-patches ALL async → checks entire component tree on every event. Overhead: ~20KB + heavy processing.
>
> **Zone-less (signals):** Direct dependency graph → only affected components update. Overhead: minimal.
>
> **Result:** Signals enable a **Zone-less Angular future** with much better performance.

---

### **Q20: How do you test signals?**

**Answer:**
> **Simple unit test:**
> ```typescript
> it('should double the count', () => {
>   const count = signal(5);
>   const double = computed(() => count() * 2);
>
>   expect(double()).toBe(10);
>
>   count.set(7);
>   expect(double()).toBe(14);
> });
> ```
>
> **With effects:**
> ```typescript
> it('should log on signal change', () => {
>   spyOn(console, 'log');
>   const count = signal(0);
>
>   effect(() => console.log(count()));
>
>   count.set(5);
>   expect(console.log).toHaveBeenCalledWith(5);
> });
> ```

---

\newpage

## 9. Common Patterns & Use Cases

### **Pattern 1: Form Validation with Signals**

```typescript
const email = signal('');
const password = signal('');

const isEmailValid = computed(() => 
  email().includes('@') && email().length > 5
);

const isPasswordValid = computed(() => 
  password().length >= 8
);

const isFormValid = computed(() =>
  isEmailValid() && isPasswordValid()
);

effect(() => {
  console.log(isFormValid() ? 'Form OK' : 'Form has errors');
});
```

### **Pattern 2: Async Data + Local Edit State**

```typescript
readonly user = toSignal(
  this.userService.getUser$(),
  { initialValue: null }
);

readonly editingUser = linkedSignal(
  () => this.user(),
  { equal: (a, b) => a?.id === b?.id }
);

saveChanges() {
  this.userService.updateUser(this.editingUser())
    .subscribe(() => {
      // After save, the source updates
      // If same user ID, editing keeps draft
      // If different user, editing resets
    });
}
```

### **Pattern 3: Sync to Local Storage**

```typescript
const theme = signal<'light' | 'dark'>('light');

effect(() => {
  localStorage.setItem('theme', theme());
  document.body.className = theme();
});

// On app init:
effect(() => {
  const saved = localStorage.getItem('theme') as 'light' | 'dark';
  if (saved) theme.set(saved);
});
```

### **Pattern 4: Prevent Infinite Effects with untracked()**

```typescript
const userId = signal(1);
const eventLog = signal<Event[]>([]);

effect(() => {
  const id = userId();  // tracked
  const now = untracked(() => new Date());  // not tracked
  
  const event = { userId: id, timestamp: now };
  eventLog.update(log => [...log, event]);
  // Won't retrigger when eventLog changes
});
```

### **Pattern 5: Dependent Signals Chain**

```typescript
const products = signal([/*...*/]);
const selectedCategory = signal('all');

const filteredProducts = computed(() =>
  selectedCategory() === 'all'
    ? products()
    : products().filter(p => p.category === selectedCategory())
);

const productCount = computed(() =>
  filteredProducts().length
);

effect(() => {
  console.log(`${productCount()} products in ${selectedCategory()}`);
});

// Change category → filtered updates → count updates → effect runs
```

---

\newpage

## 10. Do's & Don'ts — Golden Rules

### **DO ✅**

```typescript
// 1. Use signal() for local reactive state
const count = signal(0);
count.set(5);

// 2. Use computed() for derived, cached data
const double = computed(() => count() * 2);

// 3. Use effect() for side effects only
effect(() => console.log(count()));

// 4. Use toSignal() for Observables with initialValue
const user = toSignal(userService.getUser$(), 
  {initialValue: null}
);

// 5. Guard signal creation with assertNotInReactiveContext()
function createSignalHelper() {
  assertNotInReactiveContext(createSignalHelper);
  return toSignal(obs$);
}

// 6. Use linkedSignal() for editable derived data
const editCopy = linkedSignal(() => sourceUser(), 
  {equal: (a, b) => a.id === b.id}
);

// 7. Use untracked() to prevent re-runs
effect(() => {
  const val = tracked();
  const meta = untracked(() => new Date());
});

// 8. Call set/update to modify objects
user.set({...user(), name: 'Jane'});

// 9. Keep signals in components (local state)
// Store global state in services or NgRx

// 10. Create signals in constructor/ngOnInit
constructor() {
  this.data = toSignal(obs$);  // Safe
}
```

### **DON'T ❌**

```typescript
// 1. Never call set() inside computed()
const bad = computed(() => {
  count.set(5);  // ❌ infinite loop
});

// 2. Never create signals in effect() or computed()
effect(() => {
  const sig = toSignal(obs$);  // ❌ error
});

// 3. Never forget initialValue in toSignal()
const broken = toSignal(obs$);  // ❌ crash

// 4. Don't mutate objects without set()
user().name = 'John';  // ❌ no update
user.set({...user(), name: 'John'});  // ✅

// 5. Don't use effect() for state propagation
effect(() => {
  otherSignal.set(mySignal());  // ❌ use computed instead
});

// 6. Don't copy signal values manually
const copy = mySignal();  // ❌ breaks reactivity
const copy = computed(() => mySignal());  // ✅

// 7. Don't create signals globally
// Signals need injection context
export const globalCount = signal(0);  // ❌ risky

// 8. Don't track unnecessary dependencies
effect(() => {
  unneededSignal();  // ❌ causes re-runs
});

// 9. Don't use signals for complex async (use NgRx)
// Signals = local state
// NgRx = global state

// 10. Don't ignore cleanup in effects
effect((onCleanup) => {
  const timer = setTimeout(() => { }, 1000);
  // ❌ Missing: onCleanup(() => clearTimeout(timer));
});
```

---

\newpage

## 11. Performance & Best Practices

### **11.1 Performance Tips**

| Tip | Benefit |
|:----|:--------|
| Use `computed()` instead of `effect()` for derived data | Cached, deterministic |
| Use `untracked()` to prevent unnecessary re-runs | Reduces computation |
| Create signals lazily (on demand, not globally) | Smaller memory footprint |
| Use `linkedSignal()` for edit forms | Preserves drafts efficiently |
| Avoid signals inside loops/frequent functions | Prevents memory leaks |
| Use `effect()` only for true side effects | Cleaner code, less overhead |

### **11.2 Architectural Best Practices**

```typescript
// GOOD: Signals for local component state
@Component({
  selector: 'app-user-form',
  template: `...`
})
export class UserFormComponent {
  readonly email = signal('');
  readonly password = signal('');
  
  readonly isValid = computed(() => 
    this.email().length > 3 && this.password().length > 8
  );
}

// GOOD: Services bridge global state → local signals
@Injectable({providedIn: 'root'})
export class UserService {
  private readonly user$ = this.http.get('/user');
  readonly user = toSignal(this.user$, {initialValue: null});
}

// GOOD: Use NgRx for complex global state
// Use signals for components that consume it
@Component({...})
export class AppComponent {
  readonly user = toSignal(this.store.select(selectUser));
  readonly products = toSignal(this.store.select(selectProducts));
}

// AVOID: Global signals (dangerous)
export const appState = signal({...});  // ❌

// AVOID: Signals in services (unless with providedIn: 'root')
@Injectable()
export class TempService {
  data = signal([]);  // ❌ loses state on new instance
}
```

### **11.3 Memory & Cleanup**

```typescript
// Effects auto-cleanup on destroy
effect(() => {
  console.log(count());  // ✅ auto-unsubscribed
});

// Manual cleanup with onCleanup()
effect((onCleanup) => {
  const subscription = obs$.subscribe(x => { });
  
  onCleanup(() => {
    subscription.unsubscribe();  // Runs before destroy
  });
});

// toSignal() auto-unsubscribes
readonly data = toSignal(obs$, {initialValue: null});
// ✅ unsubscribed when component destroyed
```

---

\newpage

## 12. Quick Reference — Interview Checklist

Before your interview, make sure you can explain:

### **Must Know (Critical)**
- [ ] Why signals were introduced (Zone.js problem)
- [ ] The 5 core APIs: `signal()`, `computed()`, `effect()`, `linkedSignal()`, `toSignal()`
- [ ] Reactive context = tracking phase
- [ ] Signal lifecycle (Push → Pull)
- [ ] Why `toSignal()` can't run in reactive context
- [ ] Synchronous vs asynchronous (signals are sync, effects run async)

### **Should Know (Important)**
- [ ] `assertNotInReactiveContext()` purpose
- [ ] View Effect vs Root Effect timing
- [ ] `untracked()` for preventing re-runs
- [ ] `linkedSignal()` for edit forms
- [ ] Golden rules (do's & don'ts)
- [ ] Common patterns (validation, async data, storage)

### **Nice to Know (Advanced)**
- [ ] How to test signals
- [ ] Memory and cleanup strategies
- [ ] Signal comparison to RxJS subjects
- [ ] Performance optimization tips
- [ ] Architectural decisions (signals vs NgRx)

---

\newpage

## Bonus: 1-Minute Interview Openers

### **Opening Hook 1: Why Signals**
> "Angular signals solve the Zone.js performance problem by enabling fine-grained reactivity. Instead of checking the entire component tree on every event, signals create a direct producer-consumer dependency graph — so only affected components update. This is 95% more efficient and paves the way for Zone-less Angular."

### **Opening Hook 2: Reactive Context**
> "Reactive context is when Angular enters 'tracking mode' to record which signals are being read. This builds the dependency graph. That's why you can't create signals or call toSignal() inside computed() or effect() — Angular would think subscriptions are dependencies, causing infinite loops."

### **Opening Hook 3: Signal Lifecycle**
> "Signal updates happen in two phases: Push — where set() immediately marks dependents dirty, then Pull — where the next read triggers recomputation. Everything is synchronous within the same JS tick, enabling predictable fine-grained updates."

---

\newpage

## Final Tips for Interview Success

1. **Draw diagrams** when explaining reactive context and signal lifecycle
2. **Show code examples** for each concept (don't just talk theory)
3. **Mention the why** (performance, Zone-less, developer experience)
4. **Practice out loud** explaining the 5 core APIs
5. **Know the gotchas**: `toSignal()` in reactive context, `set()` in computed()
6. **Ask clarifying questions** ("Should I use signals or NgRx for this?")
7. **Be ready to code**: Live implement counter, form validation, async data
8. **Reference the problem** (Zone.js waste) when discussing benefits

---

## Resources

- **Official Docs:** https://angular.dev/guide/signals
- **RxJS Interop:** https://angular.dev/ecosystem/rxjs-interop
- **Best Practices:** https://angular.dev/guide/signals/best-practices

---

**Version:** 2.0 (Complete with toSignal, Interview Q&A)  
**Last Updated:** December 26, 2025  
**Format:** Professional Interview Preparation Guide

> **Pro Tip:** Print this document and review before your interview. Practice explaining reactive context with diagrams for maximum impact! 🚀
