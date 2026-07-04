# Topic 14 — JavaScript ES6+ (Deep Dive)

> JavaScript is the foundation of all frontend work. Senior devs must understand the engine, not just the syntax.

---

## Execution Context & Scope

```javascript
// Scope chain — each function creates a new scope:
const globalVar = "global";

function outer() {
    const outerVar = "outer";
    
    function inner() {
        const innerVar = "inner";
        console.log(globalVar); // ✅ accessible via scope chain
        console.log(outerVar);  // ✅ accessible via scope chain
        console.log(innerVar);  // ✅ own scope
    }
    
    inner();
    console.log(innerVar); // ❌ ReferenceError — not in scope
}

// var vs let vs const:
// var — function-scoped, hoisted, can re-declare
// let — block-scoped, not hoisted (TDZ), cannot re-declare
// const — block-scoped, must initialize, binding immutable (object contents can change)

for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 100); // 3 3 3 — var is shared
}
for (let i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 100); // 0 1 2 — let is block-scoped
}
```

---

## Closures

A closure is a function that **remembers** its outer variables even after the outer function has returned.

```javascript
// Classic closure — factory function:
function createCounter(start = 0) {
    let count = start; // private variable — can't be accessed from outside
    
    return {
        increment: () => ++count,
        decrement: () => --count,
        value: () => count,
        reset: () => { count = start; }
    };
}

const counter = createCounter(10);
counter.increment(); // 11
counter.increment(); // 12
counter.value();     // 12
// count is not accessible directly — true encapsulation

// Closure in React (stale closure pitfall):
function Timer() {
    const [count, setCount] = useState(0);
    
    useEffect(() => {
        const interval = setInterval(() => {
            // ❌ Stale closure — count is always 0 (captured at mount):
            setCount(count + 1);
        }, 1000);
        return () => clearInterval(interval);
    }, []); // empty deps — closure captures count=0 forever
    
    useEffect(() => {
        const interval = setInterval(() => {
            // ✅ Functional update — uses latest state:
            setCount(c => c + 1);
        }, 1000);
        return () => clearInterval(interval);
    }, []);
}
```

---

## `this` Keyword

```javascript
// this depends on how a function is CALLED, not where it's defined:

// 1. Global context:
console.log(this); // window (browser) / {} (Node.js module)

// 2. Method context:
const user = {
    name: "Randolf",
    greet() {
        console.log(this.name); // "Randolf" — this = user
    }
};

// 3. Lost context (common bug):
const greet = user.greet;
greet(); // ❌ this = undefined (strict) or window

// 4. Arrow functions — inherit this from enclosing scope:
const user2 = {
    name: "Randolf",
    greetLater() {
        setTimeout(() => {
            console.log(this.name); // ✅ "Randolf" — arrow inherits this
        }, 1000);
    }
};

// 5. Explicit binding:
function greetUser(greeting) {
    return `${greeting}, ${this.name}!`;
}
greetUser.call({ name: "Maria" }, "Hello");  // "Hello, Maria!"
greetUser.apply({ name: "Maria" }, ["Hi"]);  // "Hi, Maria!"
const boundGreet = greetUser.bind({ name: "Carlos" });
boundGreet("Hey"); // "Hey, Carlos!"
```

---

## Prototypes & Inheritance

```javascript
// Everything is an object with a prototype chain:
const arr = [1, 2, 3];
// arr → Array.prototype → Object.prototype → null

// ES5 prototype-based inheritance:
function Animal(name) {
    this.name = name;
}
Animal.prototype.speak = function() {
    return `${this.name} makes a sound`;
};

function Dog(name) {
    Animal.call(this, name); // call parent constructor
}
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;
Dog.prototype.bark = function() { return "Woof!"; };

// ES6 class (syntactic sugar over the above):
class Animal {
    #name; // private field (ES2022)
    
    constructor(name) { this.#name = name; }
    speak() { return `${this.#name} makes a sound`; }
    get name() { return this.#name; }
}

class Dog extends Animal {
    constructor(name) { super(name); }
    bark() { return "Woof!"; }
    speak() { return `${super.speak()} — specifically a bark`; }
}

const dog = new Dog("Rex");
dog instanceof Dog;    // true
dog instanceof Animal; // true — prototype chain
```

---

## Promises & Async/Await

```javascript
// Promise states: pending → fulfilled | rejected

// Creating promises:
const fetchUser = (id) => new Promise((resolve, reject) => {
    if (!id) reject(new Error("ID required"));
    setTimeout(() => resolve({ id, name: "Randolf" }), 1000);
});

// Promise chaining:
fetchUser("1")
    .then(user => fetchTransactions(user.id))
    .then(txs => console.log(txs))
    .catch(err => console.error(err))
    .finally(() => setLoading(false));

// Async/await (syntactic sugar):
async function loadDashboard(userId) {
    try {
        const user = await fetchUser(userId);
        const [transactions, budgets] = await Promise.all([
            fetchTransactions(user.id),  // parallel
            fetchBudgets(user.id)        // parallel
        ]);
        return { user, transactions, budgets };
    } catch (err) {
        console.error("Dashboard load failed:", err);
        throw err; // re-throw for caller
    }
}

// Promise combinators:
Promise.all([p1, p2, p3]);      // all must resolve; first rejection = rejects
Promise.allSettled([p1, p2]);   // waits for all; returns {status, value/reason}
Promise.race([p1, p2]);         // first to settle (resolve OR reject) wins
Promise.any([p1, p2]);          // first to RESOLVE wins (ignores rejections)
```

---

## Destructuring & Spread/Rest

```javascript
// Object destructuring:
const { id, name, email = "no-email" } = user; // default value
const { address: { city, country } } = user;   // nested
const { id: userId, ...rest } = user;           // rename + rest

// Array destructuring:
const [first, second, ...remaining] = [1, 2, 3, 4, 5];
const [, , third] = [1, 2, 3]; // skip elements

// Spread:
const updatedUser = { ...user, name: "New Name" }; // shallow copy + override
const combined = [...arr1, ...arr2, newItem];       // array merge

// Rest parameters:
function sum(...numbers) {
    return numbers.reduce((acc, n) => acc + n, 0);
}
sum(1, 2, 3, 4); // 10

// Swap variables:
let a = 1, b = 2;
[a, b] = [b, a]; // a=2, b=1
```

---

## Modules (ES Modules)

```javascript
// Named exports:
export const PI = 3.14159;
export function add(a, b) { return a + b; }
export class Calculator { /* ... */ }

// Default export (one per file):
export default function formatCurrency(amount, currency = "PHP") {
    return new Intl.NumberFormat("en-PH", { style: "currency", currency })
        .format(amount);
}

// Imports:
import formatCurrency from "./utils/currency";          // default
import { PI, add } from "./utils/math";                 // named
import { add as addition } from "./utils/math";         // rename
import * as MathUtils from "./utils/math";              // namespace
import formatCurrency, { PI } from "./utils/mixed";     // both

// Dynamic import (code splitting):
const { default: heavyModule } = await import("./heavy-module");
// Or:
const module = await import("./heavy-module");
```

---

## Array Methods (Must Know)

```javascript
const transactions = [
    { id: 1, amount: 500, type: "income", category: "Salary" },
    { id: 2, amount: 200, type: "expense", category: "Food" },
    { id: 3, amount: 150, type: "expense", category: "Transport" },
    { id: 4, amount: 1000, type: "income", category: "Freelance" },
];

// map — transform each element:
const amounts = transactions.map(t => t.amount); // [500, 200, 150, 1000]

// filter — keep matching elements:
const expenses = transactions.filter(t => t.type === "expense");

// reduce — accumulate to single value:
const totalIncome = transactions
    .filter(t => t.type === "income")
    .reduce((sum, t) => sum + t.amount, 0); // 1500

// find / findIndex:
const salary = transactions.find(t => t.category === "Salary");
const salaryIdx = transactions.findIndex(t => t.category === "Salary"); // 0

// some / every:
const hasExpenses = transactions.some(t => t.type === "expense"); // true
const allIncome  = transactions.every(t => t.type === "income");  // false

// flat / flatMap:
const nested = [[1, 2], [3, 4], [5]];
nested.flat();    // [1, 2, 3, 4, 5]
nested.flatMap(arr => arr.map(n => n * 2)); // [2, 4, 6, 8, 10]

// Group by categories (ES2024):
const byType = Object.groupBy(transactions, t => t.type);
// { income: [...], expense: [...] }

// Sort (careful — mutates original!):
const sorted = [...transactions].sort((a, b) => b.amount - a.amount);
```

---

## Event Loop & Microtasks

```javascript
// Event loop order:
// 1. Synchronous code (call stack)
// 2. Microtasks (Promise.then, queueMicrotask, MutationObserver)
// 3. Macrotasks (setTimeout, setInterval, I/O, UI events)

console.log("1 - sync");

setTimeout(() => console.log("2 - setTimeout"), 0);

Promise.resolve().then(() => console.log("3 - microtask"));

console.log("4 - sync");

// Output: 1 → 4 → 3 → 2
// Why: sync runs first, then microtasks (Promise), then macrotasks (setTimeout)

// Practical impact — understanding async React updates, debugging ordering issues
```

---

## Interview Q&A

**Q1: What is a closure? Give a practical example.**
> A closure is a function that retains access to variables from its outer scope after the outer function has returned. Practical example: a `createCounter()` factory that returns increment/decrement functions — they close over the private `count` variable, providing encapsulation.

**Q2: Explain the difference between `==` and `===`.**
> `==` performs type coercion before comparison (`"5" == 5` → true). `===` checks value AND type (`"5" === 5` → false). Always use `===` — `==` has many surprising edge cases (`null == undefined` → true, `0 == false` → true).

**Q3: What is the event loop? Why does it matter?**
> JavaScript is single-threaded. The event loop monitors the call stack and task queues. When the stack is empty, it processes microtasks (Promises) first, then macrotasks (setTimeout). This explains why `Promise.resolve().then()` runs before `setTimeout(() => {}, 0)` — Promises are microtasks.

**Q4: What is `null` vs `undefined`?**
> `undefined` means a variable has been declared but not assigned (or a function returns without a value). `null` is an intentional absence of value — you explicitly set it. `typeof undefined === "undefined"`, `typeof null === "object"` (historical bug in JS).

**Q5: What is hoisting?**
> Variable and function declarations are moved to the top of their scope during compilation. `var` declarations are hoisted and initialized to `undefined`. `function` declarations are fully hoisted. `let`/`const` are hoisted but in the Temporal Dead Zone (TDZ) — accessing them before declaration throws a `ReferenceError`.

**Q6: Explain `Promise.all` vs `Promise.allSettled`.**
> `Promise.all` rejects immediately if ANY promise rejects — use when all must succeed. `Promise.allSettled` waits for ALL promises to finish regardless of success/failure and returns an array of `{status: "fulfilled"|"rejected", value|reason}` — use when you need all results even if some fail.
