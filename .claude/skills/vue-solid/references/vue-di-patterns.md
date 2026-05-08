# Dependency Injection — Vue 3 Patterns

## Das Grundprinzip

Vue 3 DI basiert auf `provide()` / `inject()` mit typsicheren `InjectionKey`s.

```typescript
// services/container.ts
import type { InjectionKey } from 'vue'
import type { IUserService } from './interfaces/IUserService'

// Symbol als typsicherer Key
export const UserServiceKey: InjectionKey<IUserService> = Symbol('UserService')
```

## Setup in main.ts

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import { UserServiceKey } from '@/services/container'
import { UserService } from '@/services/implementations/UserService'
import { AxiosApiClient } from '@/services/implementations/AxiosApiClient'

const app = createApp(App)

// Services einmal instanziieren und global bereitstellen
const apiClient = new AxiosApiClient(import.meta.env.VITE_API_URL)
app.provide(UserServiceKey, new UserService(apiClient))

app.mount('#app')
```

## inject() in Components

```typescript
// In jedem Component der den Service braucht
import { inject } from 'vue'
import { UserServiceKey } from '@/services/container'

const userService = inject(UserServiceKey)
if (!userService) throw new Error('UserService not provided')

// Ab hier typsicher: userService ist IUserService
```

## inject() in Composables

```typescript
// composables/useUserManagement.ts
// Option A: Service als Parameter (bevorzugt — testbarer)
export function useUserManagement(userService: IUserService) { ... }

// Option B: Service intern injecten (nur wenn nötig)
export function useUserManagement() {
  const userService = inject(UserServiceKey)
  if (!userService) throw new Error('UserService not provided')
  ...
}
```

**Empfehlung: Option A** — Composable als Parameter macht Tests einfacher (kein `provide()` im Test nötig).

## Scoped Injection (Komponenten-Baum)

```typescript
// ParentComponent.vue — stellt Service für alle Kinder bereit
import { provide } from 'vue'
import { UserServiceKey } from '@/services/container'

provide(UserServiceKey, new UserService(apiClient))
```

```typescript
// ChildComponent.vue — greift auf den bereitgestellten Service zu
const userService = inject(UserServiceKey)
```

## Service Container Pattern

```typescript
// services/container.ts
export function createServiceContainer() {
  const apiClient = new AxiosApiClient(import.meta.env.VITE_API_URL)
  const userService = new UserService(apiClient)
  const authService = new AuthService(apiClient)

  return { userService, authService }
}

// main.ts
const { userService, authService } = createServiceContainer()
app.provide(UserServiceKey, userService)
app.provide(AuthServiceKey, authService)
```

## Testing mit provide/inject

```typescript
// In Composable-Tests (Option A — Service als Parameter)
import { useUserManagement } from '@/composables/useUserManagement'
import type { IUserService } from '@/services/interfaces/IUserService'

const mockService: IUserService = {
  getAll: vi.fn().mockResolvedValue([mockUser]),
  create: vi.fn(),
  delete: vi.fn(),
}

const { users, load } = useUserManagement(mockService)
```

```typescript
// In Component-Tests (provide im Test)
import { mount } from '@vue/test-utils'
import UserList from '@/components/UserList.vue'
import { UserServiceKey } from '@/services/container'

const wrapper = mount(UserList, {
  global: {
    provide: {
      [UserServiceKey as symbol]: mockService
    }
  }
})
```

## Häufige Fehler

| Fehler | Ursache | Fix |
|---|---|---|
| `inject()` gibt `undefined` | `provide()` fehlt in Vorfahren | `app.provide()` in `main.ts` |
| Typen nicht korrekt | Falscher `InjectionKey`-Typ | `InjectionKey<IMyService>` verwenden |
| `inject()` außerhalb von `setup()` | Vue-Einschränkung | Nur in `<script setup>` oder `setup()` |
| Service nicht instanziiert | `new` vergessen | Immer `new UserService(...)` in `main.ts` |
