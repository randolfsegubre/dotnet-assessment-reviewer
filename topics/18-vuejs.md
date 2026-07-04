# Topic 18 — Vue.js 3

> Vue 3 with the Composition API is increasingly popular in Philippine tech companies and startups. Easier entry point than Angular, more structured than React.

---

## Vue 3 vs Angular vs React (Quick Reference)

```
Vue 3:        <script setup> + Composition API + Pinia + Vue Router
Angular:      Component class + RxJS + Services + HttpClient (all built-in)
React:        Functional components + Hooks + external libs
```

---

## Component Basics — Composition API (script setup)

```vue
<!-- TransactionCard.vue -->
<script setup lang="ts">
import { computed, defineProps, defineEmits } from 'vue'
import type { Transaction } from '@/types'

// Props (inputs):
const props = defineProps<{
    transaction: Transaction
    isHighlighted?: boolean
}>()

// Emits (events):
const emit = defineEmits<{
    delete: [id: string]
    edit: [transaction: Transaction]
}>()

// Computed property:
const formattedAmount = computed(() =>
    new Intl.NumberFormat('en-PH', { style: 'currency', currency: 'PHP' })
        .format(props.transaction.amount)
)

const cardClass = computed(() => ({
    'card--highlighted': props.isHighlighted,
    'card--income': props.transaction.type === 'income',
    'card--expense': props.transaction.type === 'expense',
}))

function handleDelete() {
    emit('delete', props.transaction.id)
}
</script>

<template>
    <div class="card" :class="cardClass">
        <h3>{{ transaction.description }}</h3>
        <p class="amount">{{ formattedAmount }}</p>
        <span>{{ transaction.date }}</span>
        <button @click="handleDelete">Delete</button>
        <button @click="emit('edit', transaction)">Edit</button>
    </div>
</template>

<style scoped>
/* scoped — only applies to this component */
.card { border-radius: 8px; padding: 16px; border: 1px solid #ccc; }
.card--highlighted { border-color: #8b5cf6; }
</style>
```

---

## Reactivity: ref, reactive, computed, watch

```typescript
import { ref, reactive, computed, watch, watchEffect } from 'vue'

// ref — for primitives and single values:
const count = ref(0)
count.value++ // must use .value in script
// In template: {{ count }} — no .value needed

// reactive — for objects:
const user = reactive({
    name: 'Randolf',
    balance: 50000,
    accounts: [] as Account[]
})
user.name = 'Updated' // no .value needed for objects

// computed — derived state (cached):
const totalExpenses = computed(() =>
    transactions.value
        .filter(t => t.type === 'expense')
        .reduce((sum, t) => sum + t.amount, 0)
)

// watch — side effect on change:
watch(count, (newVal, oldVal) => {
    console.log(`Count: ${oldVal} → ${newVal}`)
})

// Watch multiple sources:
watch([count, user], ([newCount, newUser]) => {
    saveToLocalStorage({ count: newCount, user: newUser })
}, { deep: true }) // deep: watch nested object changes

// watchEffect — runs immediately and tracks dependencies automatically:
watchEffect(() => {
    // Runs on mount, and whenever count or user.name changes:
    document.title = `${user.name} — Balance: ${user.balance}`
})
```

---

## Lifecycle Hooks

```typescript
import {
    onMounted, onUnmounted, onUpdated,
    onBeforeMount, onBeforeUnmount
} from 'vue'

const transactions = ref<Transaction[]>([])
const loading = ref(false)

// onMounted — component is in DOM, safe to fetch:
onMounted(async () => {
    loading.value = true
    try {
        transactions.value = await fetchTransactions(userId)
    } finally {
        loading.value = false
    }
})

// onUnmounted — cleanup (subscriptions, timers):
let intervalId: number
onMounted(() => { intervalId = setInterval(refreshData, 30_000) })
onUnmounted(() => clearInterval(intervalId))

// Order: setup → onBeforeMount → onMounted → re-renders → onUpdated → onBeforeUnmount → onUnmounted
```

---

## Composables (Custom Hooks in Vue)

```typescript
// composables/useTransactions.ts
import { ref, computed } from 'vue'
import type { Transaction } from '@/types'
import { transactionApi } from '@/services/api'

export function useTransactions(userId: string) {
    const transactions = ref<Transaction[]>([])
    const loading = ref(false)
    const error = ref<string | null>(null)
    
    const totalIncome = computed(() =>
        transactions.value
            .filter(t => t.type === 'income')
            .reduce((sum, t) => sum + t.amount, 0)
    )
    
    const totalExpenses = computed(() =>
        transactions.value
            .filter(t => t.type === 'expense')
            .reduce((sum, t) => sum + t.amount, 0)
    )
    
    async function fetchAll() {
        loading.value = true
        error.value = null
        try {
            transactions.value = await transactionApi.getAll(userId)
        } catch (e) {
            error.value = 'Failed to load transactions'
        } finally {
            loading.value = false
        }
    }
    
    async function deleteTransaction(id: string) {
        await transactionApi.delete(id)
        transactions.value = transactions.value.filter(t => t.id !== id)
    }
    
    return { transactions, loading, error, totalIncome, totalExpenses, fetchAll, deleteTransaction }
}

// Usage in component:
// const { transactions, loading, totalIncome, fetchAll } = useTransactions(userId)
// onMounted(fetchAll)
```

---

## Pinia (State Management)

```typescript
// stores/authStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import type { User } from '@/types'

// Composition API style (recommended):
export const useAuthStore = defineStore('auth', () => {
    const user = ref<User | null>(null)
    const token = ref<string | null>(localStorage.getItem('token'))
    
    const isLoggedIn = computed(() => user.value !== null)
    const isAdmin = computed(() => user.value?.role === 'Admin')
    
    async function login(email: string, password: string) {
        const response = await authApi.login({ email, password })
        user.value = response.user
        token.value = response.token
        localStorage.setItem('token', response.token)
    }
    
    function logout() {
        user.value = null
        token.value = null
        localStorage.removeItem('token')
    }
    
    return { user, token, isLoggedIn, isAdmin, login, logout }
}, {
    persist: true // pinia-plugin-persistedstate — auto saves to localStorage
})

// Usage:
// const auth = useAuthStore()
// auth.login(email, password)
// auth.isLoggedIn  // computed
// auth.user.value  // reactive
```

---

## Vue Router

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/authStore'

const router = createRouter({
    history: createWebHistory(),
    routes: [
        { path: '/', redirect: '/dashboard' },
        { path: '/login', component: () => import('@/pages/LoginPage.vue') },
        {
            path: '/dashboard',
            component: () => import('@/pages/DashboardPage.vue'),
            meta: { requiresAuth: true }
        },
        {
            path: '/transactions',
            component: () => import('@/pages/TransactionsPage.vue'),
            meta: { requiresAuth: true }
        },
        {
            path: '/transactions/:id',
            component: () => import('@/pages/TransactionDetailPage.vue'),
            props: true  // pass route params as props
        },
        {
            path: '/admin',
            component: () => import('@/pages/Admin/AdminLayout.vue'),
            meta: { requiresAuth: true, requiresAdmin: true },
            children: [
                { path: 'users', component: () => import('@/pages/Admin/UsersPage.vue') }
            ]
        }
    ]
})

// Navigation guard:
router.beforeEach((to, from) => {
    const auth = useAuthStore()
    
    if (to.meta.requiresAuth && !auth.isLoggedIn)
        return { path: '/login', query: { redirect: to.fullPath } }
    
    if (to.meta.requiresAdmin && !auth.isAdmin)
        return { path: '/dashboard' }
})

export default router
```

---

## Template Directives

```vue
<template>
    <!-- v-if / v-else-if / v-else: -->
    <div v-if="loading">Loading...</div>
    <div v-else-if="error">{{ error }}</div>
    <TransactionList v-else :transactions="transactions" />
    
    <!-- v-show (CSS display toggle, element stays in DOM): -->
    <div v-show="isExpanded">Expandable content</div>
    
    <!-- v-for (always use :key!): -->
    <TransactionCard
        v-for="tx in transactions"
        :key="tx.id"
        :transaction="tx"
        @delete="handleDelete"
    />
    
    <!-- v-model (two-way binding): -->
    <input v-model="searchQuery" placeholder="Search..." />
    <select v-model="selectedType">
        <option value="">All</option>
        <option value="income">Income</option>
        <option value="expense">Expense</option>
    </select>
    
    <!-- v-bind shorthand (:), v-on shorthand (@): -->
    <button :disabled="loading" @click.prevent="handleSubmit">Submit</button>
    
    <!-- Event modifiers: -->
    <form @submit.prevent="submit">        <!-- prevent default -->
    <input @keyup.enter="search">          <!-- key filter -->
    <div @click.stop="handleClick">        <!-- stop propagation -->
    
    <!-- Slot — component composition: -->
    <BaseCard>
        <template #header><h2>Title</h2></template>
        <template #default><p>Content</p></template>
        <template #footer><button>Action</button></template>
    </BaseCard>
</template>
```

---

## Vue with TypeScript + Vite Setup

```typescript
// vite.config.ts:
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import path from 'path'

export default defineConfig({
    plugins: [vue()],
    resolve: {
        alias: { '@': path.resolve(__dirname, './src') }
    },
    server: {
        proxy: {
            '/api': { target: 'http://localhost:5106', changeOrigin: true }
        }
    }
})

// types/index.ts — shared types:
export interface Transaction {
    id: string
    amount: number
    type: 'income' | 'expense' | 'transfer'
    description: string
    date: string
    categoryId: string | null
    accountId: string
}

export interface CreateTransactionDto {
    amount: number
    type: 'income' | 'expense' | 'transfer'
    description: string
    date: string
    accountId: string
    categoryId?: string
}
```

---

## Interview Q&A

**Q1: What is the difference between `ref` and `reactive` in Vue 3?**
> `ref` wraps any value (primitive or object) in a reactive container — you access it via `.value` in script but not in templates. `reactive` creates a deep reactive proxy for objects only — no `.value` needed. Use `ref` for primitives and when you need to reassign the whole value; use `reactive` for complex objects.

**Q2: What is a composable in Vue 3?**
> A composable is a function that uses Composition API features (ref, computed, lifecycle hooks, etc.) to encapsulate and reuse stateful logic across components — equivalent to custom hooks in React. Naming convention: `useXxx`. Example: `useTransactions()`, `useAuth()`, `useForm()`.

**Q3: What is Pinia and why is it preferred over Vuex?**
> Pinia is the official Vue state management library (Vuex successor). It's simpler (no mutations, just actions), TypeScript-first, supports Vue DevTools, works with Composition API, and has a smaller bundle. Vuex required boilerplate (state/getters/mutations/actions separately) — Pinia collapses this into one clean store function.

**Q4: When would you use `v-if` vs `v-show`?**
> `v-if` removes the element from the DOM entirely — use for infrequently toggled content or when the element shouldn't be rendered at all initially. `v-show` keeps the element in DOM and toggles `display: none` — use for frequently toggled content (tabs, dropdowns) since it avoids re-rendering overhead.

**Q5: What is the Vue 3 Composition API vs Options API?**
> Options API organizes code by option type (data, methods, computed, lifecycle hooks) — easier for small components but scatters related logic. Composition API organizes code by logical concern — all related code (state, methods, lifecycle) is together. For large components and code reuse, Composition API is strongly preferred.
