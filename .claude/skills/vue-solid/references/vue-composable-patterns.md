# Composable Patterns — Vue 3

## Das Grundprinzip

Composables sind die Adapter-Schicht: Sie verbinden Vue-Reaktivität mit Business-Logic-Services. Sie enthalten **keine** Geschäftslogik — sie delegieren alles.

## Basis-Pattern

```typescript
// composables/useUsers.ts
import { ref, computed, onMounted } from 'vue'
import type { IUserService } from '@/services/interfaces/IUserService'
import type { User } from '@/models/User'

export function useUsers(userService: IUserService) {
  // Reaktiver State — nur hier, nie im Service
  const users = ref<User[]>([])
  const loading = ref(false)
  const error = ref<string | null>(null)

  // Computed aus State — kein API-Call
  const hasUsers = computed(() => users.value.length > 0)
  const adminUsers = computed(() => users.value.filter(u => u.isAdmin()))

  // Aktionen delegieren vollständig an den Service
  async function load() {
    loading.value = true
    error.value = null
    try {
      users.value = await userService.getAll()
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Fehler beim Laden'
    } finally {
      loading.value = false
    }
  }

  async function remove(id: string) {
    await userService.delete(id)
    users.value = users.value.filter(u => u.id.toString() !== id)
  }

  return { users, loading, error, hasUsers, adminUsers, load, remove }
}
```

## Lifecycle in Composables

```typescript
export function useUsers(userService: IUserService) {
  const users = ref<User[]>([])

  // Composable kann Lifecycle-Hooks registrieren
  onMounted(async () => {
    users.value = await userService.getAll()
  })

  onUnmounted(() => {
    // Cleanup: EventListener, Timer, WebSocket etc.
  })

  return { users }
}
```

## Composable mit Parametern (reaktiv)

```typescript
export function useUser(userService: IUserService, userId: Ref<string>) {
  const user = ref<User | null>(null)

  // Reagiert auf ID-Änderungen
  watch(userId, async (id) => {
    user.value = await userService.getById(id)
  }, { immediate: true })

  return { user }
}
```

## Fehlerbehandlung Pattern

```typescript
export function useAsync<T>(fn: () => Promise<T>) {
  const data = ref<T | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)

  async function execute() {
    loading.value = true
    error.value = null
    try {
      data.value = await fn()
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Unbekannter Fehler'
    } finally {
      loading.value = false
    }
  }

  return { data, loading, error, execute }
}

// Verwendung
const { data: users, loading, error, execute: loadUsers } = useAsync(
  () => userService.getAll()
)
```

## Composable Komposition

```typescript
// Composables können andere Composables nutzen
export function useUserDashboard(
  userService: IUserService,
  statsService: IStatsService
) {
  const { users, loading: usersLoading, load: loadUsers } = useUsers(userService)
  const { stats, loading: statsLoading, load: loadStats } = useStats(statsService)

  const loading = computed(() => usersLoading.value || statsLoading.value)

  async function loadAll() {
    await Promise.all([loadUsers(), loadStats()])
  }

  return { users, stats, loading, loadAll }
}
```

## Was NICHT in Composables gehört

```typescript
// ❌ Validierung im Composable
export function useCreateUser(userService: IUserService) {
  async function create(email: string) {
    if (!email.includes('@')) throw new Error('Ungültig')  // ❌ Gehört in Email-Value-Object
    return userService.create({ email })
  }
}

// ✅ Delegieren — Validierung im Service/Value Object
export function useCreateUser(userService: IUserService) {
  async function create(data: CreateUserDto) {
    return userService.create(data)  // Service/Value Object validiert
  }
}
```

```typescript
// ❌ Direkte API-Calls im Composable
export function useUsers() {
  async function load() {
    const response = await fetch('/api/users')  // ❌ Bypass des Services
    users.value = await response.json()
  }
}

// ✅ Immer über Service
export function useUsers(userService: IUserService) {
  async function load() {
    users.value = await userService.getAll()  // ✅
  }
}
```
