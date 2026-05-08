# Vue 3 SOLID — Architektur Vertiefung

## Das Drei-Schichten-Modell

```
src/
├── models/          ← Value Objects, Domain-Modelle (kein Vue)
├── services/
│   ├── interfaces/  ← IUserService, IApiClient (Interfaces)
│   └── implementations/ ← UserService, AxiosApiClient
├── validators/      ← Validierungsklassen (kein Vue)
├── composables/     ← Adapter: Vue-Reaktivität → Services
└── components/      ← Thin UI-Schicht
```

## Schicht 1: Business Logic

**Regeln ohne Ausnahme:**
- Nur TypeScript-Klassen mit Interface (Prefix `I`)
- Kein einziger `import from 'vue'`
- Dependency Injection via Constructor
- Klassen unter 50 Zeilen, eine Verantwortung

```typescript
// services/interfaces/IUserService.ts
export interface IUserService {
  getAll(): Promise<User[]>
  create(data: CreateUserDto): Promise<User>
  delete(id: string): Promise<void>
}

// services/implementations/UserService.ts
export class UserService implements IUserService {
  constructor(private readonly api: IApiClient) {}

  async getAll(): Promise<User[]> {
    const dtos = await this.api.get<UserDto[]>("/users")
    return dtos.map(User.fromDto)
  }

  async create(data: CreateUserDto): Promise<User> {
    const email = Email.create(data.email)  // Validierung im Value Object
    const dto = await this.api.post<UserDto>("/users", { ...data, email: email.toString() })
    return User.fromDto(dto)
  }

  async delete(id: string): Promise<void> {
    await this.api.delete(`/users/${id}`)
  }
}
```

## Schicht 2: Adapter (Composables)

**Composables sind Brücken — keine Logik-Schicht.**

```typescript
// composables/useUsers.ts
import { ref, computed } from 'vue'
import type { IUserService } from '@/services/interfaces/IUserService'

export function useUsers(userService: IUserService) {
  const users = ref<User[]>([])
  const loading = ref(false)
  const error = ref<string | null>(null)

  const adminCount = computed(() => users.value.filter(u => u.isAdmin()).length)

  async function load() {
    loading.value = true
    error.value = null
    try {
      users.value = await userService.getAll()  // Delegiert vollständig
    } catch (e) {
      error.value = e instanceof Error ? e.message : "Unbekannter Fehler"
    } finally {
      loading.value = false
    }
  }

  return { users, loading, error, adminCount, load }
}
```

## Schicht 3: Presentation (Components)

```vue
<!-- components/UserList.vue -->
<script setup lang="ts">
import { inject, onMounted } from 'vue'
import { UserServiceKey } from '@/services/container'
import { useUsers } from '@/composables/useUsers'

const userService = inject(UserServiceKey)
if (!userService) throw new Error('UserService not provided')

const { users, loading, error, load } = useUsers(userService)
onMounted(load)
</script>

<template>
  <div v-if="loading">Laden...</div>
  <div v-else-if="error">{{ error }}</div>
  <ul v-else>
    <li v-for="user in users" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

## Wann welche Schicht?

| Frage | Antwort |
|---|---|
| Wo kommt Validierungslogik hin? | `validators/` oder Value Object in `models/` |
| Wo kommen API-Calls hin? | `services/implementations/` via `IApiClient` |
| Wo kommt `ref()` hin? | Nur in `composables/` |
| Wo kommen `v-if`, `v-for` hin? | Nur in `components/` |
| Wo kommen Berechnungen hin? | `services/` (stateless) oder `composables/` (computed) |

## Typische Schichtverletzungen

| Code | Problem | Fix |
|---|---|---|
| `import { ref } from 'vue'` in einem Service | Vue-Abhängigkeit in Business Logic | State in Composable verwalten |
| `axios.get()` in einem Composable | API-Call nicht delegiert | In Service auslagern |
| `if (email.includes('@'))` in Component | Validierung in UI-Schicht | In Validator oder Value Object |
| Mehr als 15 Tailwind-Klassen in einem Element | Component zu fett | Subkomponente erstellen |
