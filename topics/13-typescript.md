# Topic 13 — TypeScript for Senior .NET Developers

> TypeScript is the standard for enterprise React development. Senior Full-Stack roles expect deep TS knowledge.

---

## Why TypeScript Over JavaScript

TypeScript is JavaScript + static type system. It catches errors at compile time, not runtime.

```typescript
// JavaScript — error at runtime:
function add(a, b) { return a + b; }
add("5", 3); // "53" — no error, wrong result

// TypeScript — error at compile time:
function add(a: number, b: number): number { return a + b; }
add("5", 3); // ❌ Argument of type 'string' is not assignable to 'number'
```

---

## Core Types

```typescript
// Primitives:
let name: string = "Randolf";
let age: number = 30;
let isActive: boolean = true;
let nothing: null = null;
let notSet: undefined = undefined;

// Arrays:
let scores: number[] = [100, 95, 87];
let tags: Array<string> = ["dotnet", "react"];

// Tuple — fixed-length, typed array:
let coordinate: [number, number] = [14.5995, 120.9842]; // Manila
let entry: [string, number] = ["PHP", 56.5];

// Enum:
enum TransactionType { Income = 1, Expense = 2, Transfer = 3 }
const type: TransactionType = TransactionType.Expense;

// Any (avoid!) / Unknown (safe alternative):
let dynamic: unknown = fetchData();
if (typeof dynamic === "string") {
    console.log(dynamic.toUpperCase()); // TS knows it's a string here
}

// Never — function that never returns:
function throwError(msg: string): never {
    throw new Error(msg);
}
```

---

## Interfaces vs Type Aliases

Both describe object shapes. Key differences:

```typescript
// Interface — extendable, declaration merging:
interface User {
    id: string;
    name: string;
    email: string;
}

interface AdminUser extends User {
    permissions: string[];
}

// Declaration merging (interfaces only):
interface User { avatarUrl?: string; } // adds to existing User interface

// Type alias — unions, intersections, computed types:
type UserId = string;
type Status = "active" | "inactive" | "suspended"; // union literal

type ApiResponse<T> = {
    data: T;
    message: string;
    success: boolean;
};

// Intersection (combine types):
type AdminUser = User & { permissions: string[] };

// Rule of thumb:
// - Use interface for object shapes that might be extended
// - Use type for unions, intersections, mapped types, primitives
```

---

## Generics

```typescript
// Generic function:
function first<T>(arr: T[]): T | undefined {
    return arr[0];
}
const num = first([1, 2, 3]);      // T inferred as number
const str = first(["a", "b"]);     // T inferred as string

// Generic interface:
interface Repository<T> {
    getById(id: string): Promise<T | null>;
    getAll(): Promise<T[]>;
    create(item: Omit<T, "id">): Promise<T>;
    update(id: string, item: Partial<T>): Promise<T>;
    delete(id: string): Promise<void>;
}

// Generic constraints:
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}
const user = { id: "1", name: "Randolf", age: 30 };
const name = getProperty(user, "name");   // string
const age  = getProperty(user, "age");    // number
getProperty(user, "missing");             // ❌ compile error

// Conditional types:
type NonNullable<T> = T extends null | undefined ? never : T;
type StringOrNumber = string | number | null;
type Clean = NonNullable<StringOrNumber>; // string | number
```

---

## Utility Types (Built-in)

```typescript
interface Transaction {
    id: string;
    amount: number;
    description: string;
    date: Date;
    userId: string;
}

// Partial<T> — all properties optional:
function updateTransaction(id: string, patch: Partial<Transaction>) {}
updateTransaction("1", { amount: 500 }); // only amount needed

// Required<T> — all properties required:
type CompleteTransaction = Required<Transaction>;

// Readonly<T> — no mutations:
const tx: Readonly<Transaction> = { id: "1", amount: 100, /* ... */ };
tx.amount = 200; // ❌ compile error

// Pick<T, K> — select specific properties:
type TransactionSummary = Pick<Transaction, "id" | "amount" | "date">;

// Omit<T, K> — exclude specific properties:
type CreateTransactionDto = Omit<Transaction, "id" | "userId">;

// Record<K, V> — dictionary:
type CategoryTotals = Record<string, number>;
const totals: CategoryTotals = { "Food": 5000, "Transport": 2000 };

// ReturnType<T> — extract function return type:
function fetchUser() { return { id: "1", name: "Randolf" }; }
type UserShape = ReturnType<typeof fetchUser>; // { id: string, name: string }

// Parameters<T> — extract function parameter types:
type FetchParams = Parameters<typeof fetch>; // [input: RequestInfo, init?: RequestInit]
```

---

## TypeScript with React

```typescript
// Typed component props:
interface TransactionCardProps {
    transaction: Transaction;
    onDelete: (id: string) => void;
    isHighlighted?: boolean; // optional
}

const TransactionCard: React.FC<TransactionCardProps> = ({
    transaction,
    onDelete,
    isHighlighted = false
}) => {
    return (
        <div className={isHighlighted ? "highlighted" : ""}>
            <span>{transaction.description}</span>
            <button onClick={() => onDelete(transaction.id)}>Delete</button>
        </div>
    );
};

// Typed useState:
const [transactions, setTransactions] = useState<Transaction[]>([]);
const [selectedId, setSelectedId] = useState<string | null>(null);
const [status, setStatus] = useState<"idle" | "loading" | "error">("idle");

// Typed useRef:
const inputRef = useRef<HTMLInputElement>(null);
inputRef.current?.focus(); // safe access

// Typed event handlers:
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setValue(e.target.value);
};
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
};
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    console.log(e.currentTarget.name);
};

// Typed custom hook:
function useTransactions(userId: string): {
    transactions: Transaction[];
    isLoading: boolean;
    error: string | null;
    refetch: () => void;
} {
    const [transactions, setTransactions] = useState<Transaction[]>([]);
    // ...
    return { transactions, isLoading: false, error: null, refetch: () => {} };
}
```

---

## Advanced: Discriminated Unions

```typescript
// Model API state safely — no "loading + data + error" boolean soup:
type AsyncState<T> =
    | { status: "idle" }
    | { status: "loading" }
    | { status: "success"; data: T }
    | { status: "error"; error: string };

function TransactionList({ state }: { state: AsyncState<Transaction[]> }) {
    switch (state.status) {
        case "idle":    return <p>Ready</p>;
        case "loading": return <Spinner />;
        case "success": return <List items={state.data} />;   // TS knows data exists
        case "error":   return <Error msg={state.error} />;   // TS knows error exists
    }
}

// Discriminated union for API results (mirrors C# Result<T>):
type Result<T> =
    | { ok: true; value: T }
    | { ok: false; error: string };

async function fetchTransaction(id: string): Promise<Result<Transaction>> {
    try {
        const tx = await api.get(`/transactions/${id}`);
        return { ok: true, value: tx };
    } catch (e) {
        return { ok: false, error: "Not found" };
    }
}

const result = await fetchTransaction("1");
if (result.ok) {
    console.log(result.value.amount); // TS knows value exists
} else {
    console.log(result.error);        // TS knows error exists
}
```

---

## Declaration Files & Module Types

```typescript
// When a JS library has no types:
// Create a .d.ts file or use DefinitelyTyped:
// npm install @types/lodash

// Custom declaration file (globals.d.ts):
declare global {
    interface Window {
        analytics: {
            track: (event: string, props?: Record<string, unknown>) => void;
        };
    }
}

// Augmenting existing modules:
declare module "axios" {
    interface AxiosRequestConfig {
        _retry?: boolean; // add custom property
    }
}
```

---

## tsconfig.json Key Settings

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,              // enables all strict checks
    "strictNullChecks": true,    // null/undefined must be handled
    "noImplicitAny": true,       // no untyped variables
    "noUnusedLocals": true,      // error on unused vars
    "noUnusedParameters": true,  // error on unused params
    "exactOptionalPropertyTypes": true,
    "jsx": "react-jsx",
    "baseUrl": "src",
    "paths": {
      "@components/*": ["components/*"],
      "@services/*": ["services/*"]
    }
  }
}
```

---

## TypeScript + TanStack Query v5

```typescript
// Fully typed query:
const { data, isLoading, error } = useQuery<Transaction[], Error>({
    queryKey: ["transactions", userId],
    queryFn: () => transactionService.getAll(userId),
    staleTime: 60_000,
});
// data is Transaction[] | undefined — TS enforces null checks

// Typed mutation:
const createMutation = useMutation<
    Transaction,      // return type
    Error,            // error type
    CreateTransactionDto  // variables type
>({
    mutationFn: (dto) => transactionService.create(dto),
    onSuccess: (newTx) => {
        queryClient.invalidateQueries({ queryKey: ["transactions"] });
        // newTx is Transaction — fully typed
    },
});

createMutation.mutate({ amount: 500, description: "Groceries", date: new Date() });
```

---

## Interview Q&A

**Q1: What's the difference between `interface` and `type` in TypeScript?**
> Both define object shapes. `interface` supports declaration merging and is extendable with `extends`. `type` supports unions, intersections, and computed types. Use `interface` for public API shapes, `type` for complex type algebra.

**Q2: What is a discriminated union and when would you use it?**
> A union type where each member has a common literal property (`status`, `kind`, `type`) used to distinguish them. TypeScript narrows the type in switch/if statements. Use it for API state management, command/event modeling, and eliminating impossible states.

**Q3: What does `strict: true` enable in tsconfig?**
> Enables `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, and more. Essential for production code — prevents whole categories of runtime errors.

**Q4: What is `keyof` and how is it used?**
> `keyof T` produces a union of all property names of T as string literals. Used in generic constraints: `K extends keyof T` ensures K is a valid property of T. Enables type-safe property access functions.

**Q5: How do you type an async function's return value?**
> `async function getData(): Promise<User[]>`. Always type the Promise generic parameter — this makes the caller's `await` result fully typed without inference issues.
