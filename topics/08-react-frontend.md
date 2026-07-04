# 08 — React & Frontend

> 🔗 **BudgetPH reference**: [ClientApp/src/](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.Web/ClientApp/src/) | [vite.config.js](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.Web/ClientApp/vite.config.js)

---

## Table of Contents
1. [React Hooks](#1-react-hooks)
2. [Custom Hooks](#2-custom-hooks)
3. [TanStack Query (React Query)](#3-tanstack-query-react-query)
4. [Zustand State Management](#4-zustand-state-management)
5. [react-hook-form](#5-react-hook-form)
6. [Performance Optimization](#6-performance-optimization)
7. [Vite & Build Tooling](#7-vite--build-tooling)
8. [Common Interview Questions](#8-common-interview-questions)

---

## 1. React Hooks

### useState
```jsx
import { useState } from 'react';

function AccountBalance() {
    const [balance, setBalance] = useState(0);
    const [loading, setLoading] = useState(false);

    // Functional update — safe when new state depends on old state
    const addDeposit = (amount) => {
        setBalance(prev => prev + amount); // ✅ safe
        // NOT: setBalance(balance + amount) // ❌ stale closure risk
    };

    return (
        <div>
            <p>Balance: ₱{balance.toLocaleString()}</p>
            <button onClick={() => addDeposit(1000)}>Deposit ₱1,000</button>
        </div>
    );
}
```

### useEffect
```jsx
import { useEffect, useState } from 'react';

function TransactionList({ accountId }) {
    const [transactions, setTransactions] = useState([]);

    // Effect with dependency array
    useEffect(() => {
        let cancelled = false; // cleanup flag for race conditions
        
        async function fetchTransactions() {
            const data = await transactionsService.getAll({ accountId });
            if (!cancelled) setTransactions(data);
        }
        
        fetchTransactions();
        
        // Cleanup — runs when component unmounts OR before re-run
        return () => { cancelled = true; };
    }, [accountId]); // ← re-run when accountId changes

    // Common useEffect patterns:
    // []      → runs once after mount (like componentDidMount)
    // [dep]   → runs when dep changes
    // no []   → runs after EVERY render (rarely needed)

    return <ul>{transactions.map(t => <li key={t.id}>{t.description}</li>)}</ul>;
}
```

### useCallback
Memoizes a **function reference** — prevents child components from re-rendering when the parent re-renders.
```jsx
import { useCallback, useState } from 'react';

function AccountsPage() {
    const [filter, setFilter] = useState('');

    // ✅ useCallback — stable function reference
    const handleFilterChange = useCallback((e) => {
        setFilter(e.target.value);
    }, []); // ← empty deps = created once

    // ❌ Without useCallback — new function reference every render
    // const handleFilterChange = (e) => setFilter(e.target.value);

    return (
        <div>
            <SearchInput onChange={handleFilterChange} /> {/* won't re-render */}
        </div>
    );
}
```

### useMemo
Memoizes a **computed value** — avoids expensive recalculation on every render.
```jsx
import { useMemo } from 'react';

function Dashboard({ transactions }) {
    // ✅ Only recalculates when transactions changes
    const totalExpenses = useMemo(() => {
        return transactions
            .filter(t => t.transactionType === 2) // Expense
            .reduce((sum, t) => sum + t.amount, 0);
    }, [transactions]);

    // ❌ Without useMemo — recalculates on every parent render
    // const totalExpenses = transactions.filter(...).reduce(...);

    return <p>Total Expenses: ₱{totalExpenses.toLocaleString()}</p>;
}
```

**Rule of thumb:** Don't over-use `useMemo`/`useCallback` — they have overhead. Use when:
- Computations are expensive (> few milliseconds)
- Passing functions/objects as props to `React.memo` components
- Value is a dependency of another hook

### useRef
Mutable reference that does NOT trigger re-renders.
```jsx
import { useRef, useEffect } from 'react';

function AutoFocusInput() {
    const inputRef = useRef(null);

    useEffect(() => {
        inputRef.current?.focus(); // focus input on mount
    }, []);

    return <input ref={inputRef} placeholder="Start typing..." />;
}

// Also for storing previous values
function TransactionAmount({ amount }) {
    const prevAmountRef = useRef();
    
    useEffect(() => {
        prevAmountRef.current = amount; // update AFTER render
    });
    
    const prevAmount = prevAmountRef.current;
    const changed = prevAmount !== undefined && prevAmount !== amount;
    
    return (
        <p style={{ color: changed ? 'red' : 'black' }}>
            ₱{amount.toLocaleString()}
        </p>
    );
}
```

### useContext
Shares state across component tree without prop drilling.
```jsx
// Create context
const ThemeContext = createContext({ theme: 'dark', currency: 'PHP' });

// Provider at root level
function App() {
    const [theme, setTheme] = useState('dark');
    
    return (
        <ThemeContext.Provider value={{ theme, setTheme }}>
            <Dashboard />
        </ThemeContext.Provider>
    );
}

// Consume anywhere in the tree
function TransactionRow() {
    const { theme } = useContext(ThemeContext);
    return <div className={`row ${theme}`}>...</div>;
}
```

### useReducer
For complex state transitions (like Redux pattern, local to component).
```jsx
const initialState = { 
    transactions: [], 
    loading: false, 
    error: null, 
    page: 1 
};

function reducer(state, action) {
    switch (action.type) {
        case 'FETCH_START': return { ...state, loading: true, error: null };
        case 'FETCH_SUCCESS': return { ...state, loading: false, transactions: action.payload };
        case 'FETCH_ERROR': return { ...state, loading: false, error: action.payload };
        case 'NEXT_PAGE': return { ...state, page: state.page + 1 };
        default: return state;
    }
}

function TransactionList() {
    const [state, dispatch] = useReducer(reducer, initialState);

    const fetchTransactions = async () => {
        dispatch({ type: 'FETCH_START' });
        try {
            const data = await transactionsService.getAll({ page: state.page });
            dispatch({ type: 'FETCH_SUCCESS', payload: data });
        } catch (err) {
            dispatch({ type: 'FETCH_ERROR', payload: err.message });
        }
    };

    return <>{state.loading ? <Spinner /> : <Table data={state.transactions} />}</>;
}
```

---

## 2. Custom Hooks

Custom hooks extract reusable logic — they must start with `use`.

```jsx
// Custom hook for paginated data fetching
function usePaginatedTransactions(filters = {}) {
    const [page, setPage] = useState(1);
    const pageSize = 20;

    const { data, isLoading, error } = useQuery({
        queryKey: ['transactions', page, filters],
        queryFn: () => transactionsService.getAll({ page, pageSize, ...filters }),
        placeholderData: keepPreviousData, // TanStack Query v5
    });

    return {
        transactions: data?.items ?? [],
        total: data?.total ?? 0,
        page,
        pageSize,
        totalPages: Math.ceil((data?.total ?? 0) / pageSize),
        isLoading,
        error,
        goToPage: setPage,
        nextPage: () => setPage(p => Math.min(p + 1, Math.ceil((data?.total ?? 0) / pageSize))),
        prevPage: () => setPage(p => Math.max(p - 1, 1)),
    };
}

// Custom hook for form with API submission
function useCreateTransaction() {
    const queryClient = useQueryClient();
    
    const mutation = useMutation({
        mutationFn: (data) => transactionsService.create(data),
        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: ['transactions'] });
            queryClient.invalidateQueries({ queryKey: ['dashboard'] });
        },
    });

    return mutation;
}

// Usage in component
function AddTransactionForm() {
    const { transactions, isLoading, page, goToPage } = usePaginatedTransactions();
    const createMutation = useCreateTransaction();
    
    const handleSubmit = async (data) => {
        await createMutation.mutateAsync(data);
    };
}
```

---

## 3. TanStack Query (React Query)

TanStack Query (v5) manages **server state** — fetching, caching, synchronization.

### 🔗 BudgetPH uses TanStack Query throughout all pages

```jsx
import { useQuery, useMutation, useQueryClient, keepPreviousData } from '@tanstack/react-query';

// Basic query
function AccountsPage() {
    const { data: accounts, isLoading, isError, error, refetch } = useQuery({
        queryKey: ['accounts'],          // ← cache key
        queryFn: () => accountsService.getAll(),
        staleTime: 1000 * 60 * 5,       // 5 minutes — don't refetch if recent
        gcTime: 1000 * 60 * 10,         // 10 minutes — keep in cache (was cacheTime in v4)
        retry: 3,                        // retry failed requests 3 times
        refetchOnWindowFocus: true,     // refetch when tab regains focus
    });

    if (isLoading) return <Spinner />;
    if (isError) return <Error message={error.message} />;
    return <AccountList accounts={accounts} />;
}

// Mutation (create/update/delete)
function AddAccountModal({ onClose }) {
    const queryClient = useQueryClient();
    
    const mutation = useMutation({
        mutationFn: (data) => accountsService.create(data),
        onMutate: async (newAccount) => {
            // Optimistic update
            await queryClient.cancelQueries({ queryKey: ['accounts'] });
            const previous = queryClient.getQueryData(['accounts']);
            queryClient.setQueryData(['accounts'], old => [...old, { ...newAccount, id: 'temp' }]);
            return { previous }; // return snapshot for rollback
        },
        onError: (err, variables, context) => {
            // Rollback on error
            queryClient.setQueryData(['accounts'], context.previous);
        },
        onSettled: () => {
            // Always refetch after success or error
            queryClient.invalidateQueries({ queryKey: ['accounts'] });
        },
        onSuccess: () => {
            onClose();
        }
    });

    return (
        <form onSubmit={handleSubmit(data => mutation.mutate(data))}>
            {mutation.isPending && <Spinner />}
            {mutation.isError && <Alert>{mutation.error.message}</Alert>}
            <button type="submit" disabled={mutation.isPending}>Save</button>
        </form>
    );
}

// Dependent queries
function TransactionDetails({ transactionId }) {
    const { data: transaction } = useQuery({
        queryKey: ['transaction', transactionId],
        queryFn: () => transactionsService.getById(transactionId),
    });

    const { data: account } = useQuery({
        queryKey: ['account', transaction?.accountId],
        queryFn: () => accountsService.getById(transaction.accountId),
        enabled: !!transaction?.accountId, // ← only runs when accountId is available
    });
}
```

### Query Invalidation Strategy

```jsx
// After creating a transaction, invalidate related queries
queryClient.invalidateQueries({ queryKey: ['transactions'] });
queryClient.invalidateQueries({ queryKey: ['accounts'] });     // balance changed
queryClient.invalidateQueries({ queryKey: ['dashboard'] });    // dashboard stats
queryClient.invalidateQueries({ queryKey: ['budgets'] });      // budget tracking

// Invalidate a specific account
queryClient.invalidateQueries({ queryKey: ['account', accountId] });

// Prefetch on hover for instant feel
const prefetch = () => queryClient.prefetchQuery({
    queryKey: ['transaction', id],
    queryFn: () => transactionsService.getById(id),
});
```

---

## 4. Zustand State Management

Zustand manages **client state** (auth, UI preferences, etc.).

### 🔗 BudgetPH Auth Store

```js
// src/store/authStore.js
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

const useAuthStore = create(
    persist(
        (set, get) => ({
            // State
            token: null,
            refreshToken: null,
            expiresAt: null,
            user: null,

            // Actions
            setAuth: (authResponse) => set({
                token: authResponse.accessToken,
                refreshToken: authResponse.refreshToken,
                expiresAt: authResponse.expiresAt,
                user: authResponse.user,
            }),

            clearAuth: () => set({
                token: null, refreshToken: null, expiresAt: null, user: null,
            }),

            updateUser: (user) => set({ user }),

            isTokenExpired: () => {
                const { expiresAt } = get();
                if (!expiresAt) return true;
                return new Date(expiresAt) <= new Date();
            },
        }),
        { name: 'budgetph-auth' } // localStorage key
    )
);

// Usage in component
function Navbar() {
    const { user, clearAuth } = useAuthStore();
    
    const handleLogout = () => {
        clearAuth();
        navigate('/login');
    };
    
    return <div>Welcome, {user?.firstName}! <button onClick={handleLogout}>Logout</button></div>;
}

// Usage in API interceptor (Axios)
const token = useAuthStore.getState().token; // outside components — use getState()
```

### Zustand vs Context API

| | Zustand | Context API |
|---|---|---|
| Re-renders | Only components subscribed to changed slice | ALL consumers re-render on any change |
| Boilerplate | Minimal | Moderate (Provider, value memo) |
| DevTools | ✅ Redux DevTools compatible | ❌ None |
| Async actions | ✅ Direct in store | Need external patterns |
| Bundle size | Tiny (~1KB) | Zero (built-in) |

---

## 5. react-hook-form

### 🔗 BudgetPH uses react-hook-form for all forms

```jsx
import { useForm } from 'react-hook-form';

function LoginForm() {
    const {
        register,           // connects input to form
        handleSubmit,       // wraps submit handler with validation
        formState: { errors, isSubmitting },
        setError,           // set server errors
        reset,              // reset to initial values
        watch,              // watch field values
        setValue,           // programmatically set a field
        getValues,          // read field values without subscription
    } = useForm({
        defaultValues: { email: '', password: '' },
        mode: 'onBlur',   // validate: 'onChange' | 'onBlur' | 'onSubmit' | 'all'
    });

    const onSubmit = async (data) => {
        try {
            await authService.login(data);
        } catch (err) {
            // Set server-side validation error on specific field
            setError('email', { 
                type: 'server', 
                message: 'Invalid email or password' 
            });
        }
    };

    return (
        <form onSubmit={handleSubmit(onSubmit)}>
            <input
                {...register('email', {
                    required: 'Email is required',
                    pattern: {
                        value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
                        message: 'Invalid email address'
                    }
                })}
                type="email"
            />
            {errors.email && <span>{errors.email.message}</span>}

            <input
                {...register('password', {
                    required: 'Password is required',
                    minLength: { value: 8, message: 'Minimum 8 characters' }
                })}
                type="password"
            />
            {errors.password && <span>{errors.password.message}</span>}

            <button type="submit" disabled={isSubmitting}>
                {isSubmitting ? 'Logging in...' : 'Login'}
            </button>
        </form>
    );
}
```

---

## 6. Performance Optimization

### React.memo — Prevent Unnecessary Re-renders

```jsx
// Wrap component in React.memo — only re-renders if props changed
const AccountCard = React.memo(function AccountCard({ account }) {
    return (
        <div className="card">
            <h3>{account.name}</h3>
            <p>Balance: ₱{account.balance.toLocaleString()}</p>
        </div>
    );
});

// With custom comparison
const TransactionRow = React.memo(
    function TransactionRow({ transaction, onDelete }) {
        return <tr>...</tr>;
    },
    (prevProps, nextProps) => {
        // Return true if props are equal (skip re-render)
        return prevProps.transaction.id === nextProps.transaction.id &&
               prevProps.transaction.amount === nextProps.transaction.amount;
    }
);
```

### Code Splitting with lazy + Suspense

```jsx
import { lazy, Suspense } from 'react';

// Lazy load heavy pages — only downloaded when navigated to
const ReportsPage = lazy(() => import('./pages/Reports/ReportsPage'));
const InvestmentsPage = lazy(() => import('./pages/Investments/InvestmentsPage'));

function App() {
    return (
        <Suspense fallback={<div className="loading-spinner">Loading...</div>}>
            <Routes>
                <Route path="/reports" element={<ReportsPage />} />
                <Route path="/investments" element={<InvestmentsPage />} />
            </Routes>
        </Suspense>
    );
}
```

### Virtualization for Long Lists

```jsx
// Use react-window or react-virtual for 1000+ item lists
import { FixedSizeList } from 'react-window';

function TransactionList({ transactions }) {
    const Row = ({ index, style }) => (
        <div style={style}>
            <TransactionRow transaction={transactions[index]} />
        </div>
    );

    return (
        <FixedSizeList
            height={600}
            itemCount={transactions.length}
            itemSize={50}
            width="100%"
        >
            {Row}
        </FixedSizeList>
    );
}
```

---

## 7. Vite & Build Tooling

### 🔗 BudgetPH Vite Config

From [vite.config.js](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.Web/ClientApp/vite.config.js):
```js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
    plugins: [react()],
    server: {
        port: 3000,
        proxy: {
            // Proxy all /api requests to the .NET backend
            '/api': {
                target: 'http://localhost:5106',
                changeOrigin: true,
                secure: false,
            }
        }
    }
});
```

### Why Vite Over CRA (Create React App)?

| | Vite | CRA (Webpack) |
|---|---|---|
| Dev server start | Near instant (ESM native) | Slow (bundles everything first) |
| HMR (Hot Module Reload) | Instant | Slow |
| Build speed | Fast (Rollup) | Slow |
| Config | Simple | Complex ejection needed |

---

## 8. Common Interview Questions

**Q1: What is the difference between `useCallback` and `useMemo`?**

> `useCallback` memoizes a **function** — returns the same function reference if dependencies haven't changed. `useMemo` memoizes a **value** — returns the same computed value if dependencies haven't changed. Both prevent unnecessary recalculation/recreation on re-renders, but `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.

**Q2: When does `useEffect` run without a dependency array vs with `[]` vs with `[dep]`?**

> No array `useEffect(() => {...})` — runs after every render. Empty array `useEffect(() => {...}, [])` — runs once after mount. With dependencies `useEffect(() => {...}, [dep])` — runs after mount and whenever `dep` changes.

**Q3: What is the difference between controlled and uncontrolled components?**

> **Controlled**: React state is the single source of truth; every input change calls `setState`; form data lives in React state. **Uncontrolled**: DOM itself holds the state; you use `ref` to read values on submit; more like traditional HTML forms. `react-hook-form` uses an optimized mix — it uses uncontrolled inputs internally but provides a controlled API.

**Q4: What is the Context API and when would you use Zustand instead?**

> Context API is built-in for sharing state without prop drilling. But every Context consumer re-renders when the context value changes — even if only one part changed. Zustand solves this with **granular subscriptions** — components only re-render when the specific slice of state they subscribe to changes. Use Context for simple, low-frequency-update state (theme, locale). Use Zustand for complex, frequently-updating state (auth, filters, UI state).

**Q5: Explain the TanStack Query "staleTime" and "gcTime".**

> `staleTime`: how long data is considered fresh. If you navigate back to a page within staleTime, the cached data is shown without a network request. `gcTime` (formerly `cacheTime`): how long inactive query data stays in the cache before garbage collection. Default: staleTime=0 (immediately stale), gcTime=5min.

**Q6: What is a race condition in React and how do you handle it?**

> When multiple async operations are in flight, older results can arrive after newer ones, overwriting fresh data with stale data. Fix: use a cleanup flag in `useEffect` (`let cancelled = false; return () => { cancelled = true; }`), or use `AbortController` with `fetch`, or use TanStack Query (which handles this automatically).
