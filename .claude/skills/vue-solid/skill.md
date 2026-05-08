---
name: vue-solid
description: |
  SOLID principles for Vue 3 + TypeScript projects. Applies SOLID to business logic (Services, Models, Validators) implemented as TypeScript classes with interfaces. Uses Composables ONLY as adapter layer between Vue reactivity and business logic. Components remain thin, delegating to composables. Enforces clean separation: Business Logic (classes, no Vue) → Adapter (composables) → Presentation (components).
---

# Vue 3 SOLID Architecture

You are now operating as a senior Vue 3 + TypeScript engineer who writes production-grade code following SOLID principles, Clean Code practices, and professional software design patterns.

## Core Principle

**SOLID applies to BUSINESS LOGIC, not to Vue's presentation layer.**

Vue 3 projects have THREE distinct layers:
1. **Business Logic Layer** (services/, models/, validators/) - SOLID principles apply 100%
2. **Adapter Layer** (composables/) - Bridges Vue reactivity with business logic
3. **Presentation Layer** (components/) - Vue-specific UI code

## Architecture Rules

### Layer 1: Business Logic (SOLID 100%)

**Location:** `src/services/`, `src/models/`, `src/validators/`

**Rules:**
- ALWAYS use TypeScript classes
- ALWAYS define interfaces (prefix with `I`)
- NEVER import from 'vue' (no ref, computed, inject, etc.)
- Apply all SOLID principles strictly
- Use Dependency Injection via constructor
- Keep classes small (<50 lines preferred)
- One responsibility per class

**Example:**
```typescript
// services/interfaces/IUserService.ts
export interface IUserService {
  getAll(): Promise<User[]>
  getById(id: string): Promise<User | null>
  create(data: CreateUserDto): Promise<User>
  update(id: string, data: UpdateUserDto): Promise<User>
  delete(id: string): Promise<void>
}

// services/implementations/UserService.ts
export class UserService implements IUserService {
  constructor(
    private apiClient: IApiClient,
    private validator: IValidator<User>
  ) {}
  
  async getAll(): Promise<User[]> {
    const dtos = await this.apiClient.get<UserDto[]>('/users')
    return dtos.map(User.fromDto)
  }
  
  // ... other methods
}
```

### Layer 2: Adapter (Composables)

**Location:** `src/composables/`

**Purpose:** Bridge between Vue reactivity and business logic

**Rules:**
- Functions that accept service instances (via inject or parameter)
- Manage Vue-specific state (ref, computed, watch)
- Handle loading/error states
- Manage lifecycle (onMounted, onUnmounted)
- **MINIMAL business logic** - delegate everything to services
- Never implement validation, domain rules, or complex algorithms

**Example:**
```typescript
// composables/useUserManagement.ts
import { ref, computed } from 'vue'
import type { IUserService } from '@/services/interfaces/IUserService'
import type { User } from '@/models/User'

export function useUserManagement(userService: IUserService) {
  // Vue-specific reactive state
  const users = ref<User[]>([])
  const loading = ref(false)
  const error = ref<string | null>(null)
  
  // Vue-specific computed properties
  const hasUsers = computed(() => users.value.length > 0)
  const adminUsers = computed(() => users.value.filter(u => u.isAdmin()))
  
  // Functions delegate to service
  async function loadUsers() {
    loading.value = true
    error.value = null
    
    try {
      users.value = await userService.getAll() // ← Delegate to service
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Unknown error'
      console.error('Failed to load users:', e)
    } finally {
      loading.value = false
    }
  }
  
  async function createUser(data: CreateUserDto) {
    loading.value = true
    error.value = null
    
    try {
      const user = await userService.create(data) // ← Delegate to service
      users.value.push(user)
      return user
    } catch (e) {
      error.value = 'Failed to create user'
      throw e
    } finally {
      loading.value = false
    }
  }
  
  return {
    // State
    users,
    loading,
    error,
    hasUsers,
    adminUsers,
    
    // Actions
    loadUsers,
    createUser
  }
}
```

### Layer 3: Presentation (Components)

**Location:** `src/components/`

**Rules:**
- Vue SFC files (.vue)
- Inject services via `inject()`
- Use composables for reactive state
- Handle user events
- **NO business logic** - delegate to composables/services
- **NO validation logic** - services handle validation
- **NO API calls** - services handle API communication

**Example:**
```vue
<!-- components/user/UserList.vue -->
<script setup lang="ts">
import { inject, onMounted } from 'vue'
import { UserServiceKey } from '@/services/container'
import { useUserManagement } from '@/composables/useUserManagement'
import UserCard from './UserCard.vue'

// Inject service
const userService = inject(UserServiceKey)
if (!userService) {
  throw new Error('UserService not provided')
}

// Use composable
const { 
  users, 
  loading, 
  error, 
  loadUsers, 
  createUser 
} = useUserManagement(userService)

// Component only handles lifecycle
onMounted(() => {
  loadUsers()
})
</script>

<template>
  <div class="user-list">
    <h2>Users</h2>
    
    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else-if="users.length === 0" class="empty">No users found</div>
    
    <div v-else class="user-grid">
      <UserCard
        v-for="user in users"
        :key="user.id"
        :user="user"
      />
    </div>
  </div>
</template>
```

## SOLID Principles in Practice

### Single Responsibility Principle (SRP)

Each class/module has ONE reason to change:

```typescript
// ✅ GOOD - Each class has one responsibility
class Email {
  // Only responsible for email validation
  static create(value: string): Email { ... }
}

class UserService {
  // Only responsible for user operations
  async getAll(): Promise<User[]> { ... }
}

class UserValidator {
  // Only responsible for user validation
  validate(user: User): ValidationResult { ... }
}

// ❌ BAD - God object with multiple responsibilities
class UserManager {
  validateEmail(email: string) { ... }
  saveToDatabase(user: User) { ... }
  sendWelcomeEmail(user: User) { ... }
  generateReport() { ... }
}
```

### Open/Closed Principle (OCP)

Open for extension, closed for modification:

```typescript
// ✅ GOOD - Strategy pattern
interface INotificationService {
  notify(message: string): void
}

class ToastNotificationService implements INotificationService {
  notify(message: string) { /* toast */ }
}

class EmailNotificationService implements INotificationService {
  notify(message: string) { /* email */ }
}

// Add new notification types without modifying existing code

// ❌ BAD - Must modify for each new type
class NotificationService {
  notify(message: string, type: 'toast' | 'email' | 'sms') {
    if (type === 'toast') { /* ... */ }
    else if (type === 'email') { /* ... */ }
    else if (type === 'sms') { /* ... */ }
  }
}
```

### Liskov Substitution Principle (LSP)

Subtypes must be substitutable for their base types:

```typescript
// ✅ GOOD - All implementations are interchangeable
interface IApiClient {
  get<T>(url: string): Promise<T>
  post<T>(url: string, data: any): Promise<T>
}

class AxiosApiClient implements IApiClient {
  async get<T>(url: string): Promise<T> { /* axios */ }
  async post<T>(url: string, data: any): Promise<T> { /* axios */ }
}

class MockApiClient implements IApiClient {
  async get<T>(url: string): Promise<T> { /* mock */ }
  async post<T>(url: string, data: any): Promise<T> { /* mock */ }
}

// UserService works with BOTH - they're substitutable
const service = new UserService(new AxiosApiClient())
const testService = new UserService(new MockApiClient())
```

### Interface Segregation Principle (ISP)

Clients shouldn't depend on interfaces they don't use:

```typescript
// ✅ GOOD - Segregated interfaces
interface IUserReader {
  getAll(): Promise<User[]>
  getById(id: string): Promise<User | null>
}

interface IUserWriter {
  create(user: CreateUserDto): Promise<User>
  update(id: string, user: UpdateUserDto): Promise<User>
  delete(id: string): Promise<void>
}

// Full service combines both
interface IUserService extends IUserReader, IUserWriter {}

// Composables can depend on only what they need
function useUserList(reader: IUserReader) {
  // Only needs read access
}

// ❌ BAD - Fat interface
interface IUserService {
  getAll(): Promise<User[]>
  create(user: User): Promise<User>
  exportToCsv(): Promise<string>
  sendEmail(userId: string): Promise<void>
  generateReport(): Promise<Report>
}
```

### Dependency Inversion Principle (DIP)

Depend on abstractions, not concretions:

```typescript
// ✅ GOOD - Depends on interface
class UserService implements IUserService {
  constructor(
    private apiClient: IApiClient,        // ← Interface
    private validator: IValidator<User>   // ← Interface
  ) {}
}

// ❌ BAD - Depends on concrete implementation
class UserService {
  constructor() {
    this.apiClient = new AxiosApiClient() // ← Concrete class
    this.validator = new UserValidator()  // ← Concrete class
  }
}
```

## Dependency Injection Pattern

### Setup in main.ts

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import { createServiceContainer } from './services/container'

const app = createApp(App)
const container = createServiceContainer()

// Provide services globally
app.provide('userService', container.get('UserService'))
app.provide('notificationService', container.get('NotificationService'))

app.mount('#app')
```

### Type-Safe Injection Keys

```typescript
// services/container.ts
import type { InjectionKey } from 'vue'
import type { IUserService } from './interfaces/IUserService'
import type { INotificationService } from './interfaces/INotificationService'

export const UserServiceKey: InjectionKey<IUserService> = Symbol('UserService')
export const NotificationServiceKey: InjectionKey<INotificationService> = Symbol('NotificationService')

export function createServiceContainer() {
  // Create instances
  const apiClient = new AxiosApiClient(import.meta.env.VITE_API_URL)
  const notificationService = new ToastNotificationService()
  const userService = new UserService(apiClient)
  
  return {
    get(key: string) {
      const services = {
        UserService: userService,
        NotificationService: notificationService
      }
      return services[key]
    }
  }
}
```

### Usage in Components

```typescript
// Component
import { inject } from 'vue'
import { UserServiceKey } from '@/services/container'

const userService = inject(UserServiceKey)
if (!userService) {
  throw new Error('UserService not provided!')
}
```

## Value Objects

Value Objects encapsulate validation and domain logic:

```typescript
// models/Email.ts
export class Email {
  private readonly value: string

  private constructor(value: string) {
    this.value = value
  }

  static create(value: string): Email {
    if (!Email.isValid(value)) {
      throw new Error(`Invalid email: ${value}`)
    }
    return new Email(value)
  }

  private static isValid(email: string): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
    return emailRegex.test(email)
  }

  toString(): string {
    return this.value
  }

  equals(other: Email): boolean {
    return this.value === other.value
  }
}

// Usage
const email = Email.create('user@example.com') // ✅ Valid
const invalid = Email.create('not-an-email') // ❌ Throws error
```

## Domain Models

```typescript
// models/User.ts
import { Email } from './Email'
import { UserId } from './UserId'

export interface UserDto {
  id: string
  name: string
  email: string
  role: 'admin' | 'user'
  created_at: string
}

export class User {
  private constructor(
    public readonly id: UserId,
    public readonly name: string,
    public readonly email: Email,
    public readonly role: 'admin' | 'user',
    public readonly createdAt: Date
  ) {}

  static create(props: Omit<UserDto, 'id' | 'created_at'>): User {
    return new User(
      UserId.generate(),
      props.name,
      Email.create(props.email),
      props.role,
      new Date()
    )
  }

  static fromDto(dto: UserDto): User {
    return new User(
      UserId.create(dto.id),
      dto.name,
      Email.create(dto.email),
      dto.role,
      new Date(dto.created_at)
    )
  }

  isAdmin(): boolean {
    return this.role === 'admin'
  }

  toDto(): UserDto {
    return {
      id: this.id.toString(),
      name: this.name,
      email: this.email.toString(),
      role: this.role,
      created_at: this.createdAt.toISOString()
    }
  }
}
```

## File Structure

```
src/
├── main.ts                           # DI Container setup
├── App.vue
├── models/                           # Domain models
│   ├── User.ts
│   ├── Email.ts                      # Value object
│   └── UserId.ts                     # Value object
├── services/
│   ├── container.ts                  # DI setup + Injection keys
│   ├── interfaces/
│   │   ├── IUserService.ts
│   │   ├── IApiClient.ts
│   │   └── INotificationService.ts
│   └── implementations/
│       ├── UserService.ts
│       ├── AxiosApiClient.ts
│       └── ToastNotificationService.ts
├── composables/                      # Vue adapters
│   ├── useUserManagement.ts
│   ├── useNotifications.ts
│   └── useAuth.ts
├── validators/
│   ├── IValidator.ts
│   └── UserValidator.ts
├── stores/                           # Pinia stores
│   └── userStore.ts
└── components/
    └── user/
        ├── UserList.vue
        └── UserCard.vue
```

## Anti-Patterns to Avoid

### ❌ Business Logic in Composables

```typescript
// ❌ BAD
export function useUsers() {
  const users = ref([])
  
  async function createUser(email: string) {
    // ❌ Validation in composable
    if (!email.includes('@')) {
      throw new Error('Invalid email')
    }
    
    // ❌ Business rule in composable
    if (users.value.length >= 100) {
      throw new Error('User limit reached')
    }
    
    // ❌ Direct API call
    const response = await fetch('/api/users', {
      method: 'POST',
      body: JSON.stringify({ email })
    })
  }
}

// ✅ GOOD
export function useUserManagement(userService: IUserService) {
  const users = ref<User[]>([])
  
  async function createUser(email: string) {
    // ✅ Delegate everything to service
    const user = await userService.create({ email })
    users.value.push(user)
  }
}
```

### ❌ Vue imports in Business Logic

```typescript
// ❌ BAD
import { ref } from 'vue'

export class UserService {
  private users = ref([]) // ❌ Vue dependency in service
}

// ✅ GOOD
export class UserService {
  // No Vue dependencies
  async getAll(): Promise<User[]> {
    return this.apiClient.get('/users')
  }
}
```

### ❌ God Components

```vue
<!-- ❌ BAD -->
<script setup>
import axios from 'axios'

const users = ref([])
const loading = ref(false)

async function loadUsers() {
  loading.value = true
  const response = await axios.get('/api/users')
  users.value = response.data
  loading.value = false
}

function validateEmail(email) {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)
}

async function createUser(data) {
  if (!validateEmail(data.email)) {
    alert('Invalid email')
    return
  }
  // ... more logic
}

onMounted(loadUsers)
</script>

<!-- ✅ GOOD -->
<script setup>
import { inject, onMounted } from 'vue'
import { UserServiceKey } from '@/services/container'
import { useUserManagement } from '@/composables/useUserManagement'

const userService = inject(UserServiceKey)!
const { users, loading, loadUsers } = useUserManagement(userService)

onMounted(loadUsers)
</script>
```

## Testing Strategy

### Service Tests (No Vue)

```typescript
// __tests__/UserService.spec.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { UserService } from '@/services/implementations/UserService'
import { MockApiClient } from '@/services/implementations/MockApiClient'

describe('UserService', () => {
  let apiClient: MockApiClient
  let userService: UserService

  beforeEach(() => {
    apiClient = new MockApiClient()
    userService = new UserService(apiClient)
  })

  it('should get all users', async () => {
    const users = await userService.getAll()
    
    expect(users).toHaveLength(2)
    expect(users[0]).toBeInstanceOf(User)
  })

  it('should create a user', async () => {
    const user = await userService.create({
      name: 'Alice',
      email: 'alice@example.com',
      role: 'user'
    })
    
    expect(user.name).toBe('Alice')
    expect(user.email.toString()).toBe('alice@example.com')
  })
})
```

### Composable Tests

```typescript
// __tests__/useUserManagement.spec.ts
import { describe, it, expect, vi } from 'vitest'
import { useUserManagement } from '@/composables/useUserManagement'
import type { IUserService } from '@/services/interfaces/IUserService'

describe('useUserManagement', () => {
  it('should load users', async () => {
    const mockService: IUserService = {
      getAll: vi.fn().mockResolvedValue([mockUser]),
      // ... other methods
    }

    const { users, loadUsers } = useUserManagement(mockService)
    
    await loadUsers()
    
    expect(users.value).toHaveLength(1)
    expect(mockService.getAll).toHaveBeenCalledOnce()
  })
})
```

## Best Practices Summary

### ✅ DO:

1. **Use TypeScript classes for business logic**
   - Services, Models, Validators
   - Always with interfaces

2. **Keep composables thin**
   - Only Vue reactivity management
   - Delegate to services

3. **Value Objects for domain primitives**
   - Email, UserId, Money
   - Validation in constructor

4. **Dependency Injection everywhere**
   - Constructor injection in classes
   - provide/inject in Vue

5. **Small, focused classes**
   - Single Responsibility
   - Under 50 lines preferred

### ❌ DON'T:

1. **No business logic in components**
2. **No Vue imports in services/models**
3. **No validation in composables**
4. **No `any` types**
5. **No God objects**

## When to Use This Skill

ALWAYS use this skill when:
- Writing any Vue 3 + TypeScript code
- Creating services, models, or validators
- Refactoring existing code
- Planning architecture
- Reviewing code quality
- Creating tests
- Making design decisions

For detailed guidance on specific topics, see the reference documentation:
- [Architecture Patterns](references/vue-architecture.md)
- [Dependency Injection](references/vue-di-patterns.md)
- [Composable Patterns](references/vue-composable-patterns.md)
- [Testing Strategies](references/vue-testing.md)
- [API Integration](references/vue-api-integration.md)

## Layer 4: Styling & UI (Tailwind CSS)

**Principles:**
- **Utility-First:** No custom CSS/`<style>` blocks unless absolutely necessary for complex animations.
- **Mobile-First:** Write classes for mobile by default. Use `md:`, `lg:`, etc., only to expand for larger screens.
- **Single Responsibility (Styles):** If a component's class list exceeds 10-15 classes, evaluate if it should be split into smaller sub-components.

**Rules:**
1. **Layout Strategy:** 
   - Mobile: Vertical stacks (`flex-col`, `grid-cols-1`), full width (`w-full`), and touch-friendly margins.
   - Desktop (`md:` and up): Multi-column layouts (`md:grid-cols-12`, `md:flex-row`), constrained widths (`max-w-7xl`).
2. **Touch Targets:** Interactive elements (buttons, links, inputs) must have a minimum size of `h-12` (48px) for mobile usability.
3. **Spacing & Typography:**
   - Use a consistent scale: `p-4` (mobile) → `md:p-8` (desktop).
   - Use responsive text: `text-base` (mobile) → `md:text-lg` (desktop).
4. **Clean Templates:** 
   - Use Tailwind's grouping and peer modifiers for state changes (hover, focus).
   - Keep templates readable; use line breaks between long class lists if necessary.

**Example (Responsive & SOLID):**
```vue
<!-- components/rules/RuleCard.vue (Presentation Layer) -->
<template>
  <div class="flex flex-col gap-4 p-4 border rounded-xl bg-white shadow-sm 
              md:flex-row md:gap-6 md:p-6 hover:shadow-md transition-shadow">
    <div class="h-12 w-12 flex-shrink-0 bg-blue-100 text-blue-600 rounded-lg flex items-center justify-center">
      <slot name="icon" />
    </div>
    <div class="flex-1">
      <h3 class="text-lg font-bold text-gray-900 md:text-xl">{{ title }}</h3>
      <p class="mt-1 text-sm text-gray-600 leading-relaxed md:text-base">
        <slot />
      </p>
    </div>
  </div>
</template>
```