# Testing — Vue 3 + TypeScript

## Test-Stack

| Schicht | Framework |
|---|---|
| Models / Services | Vitest (reines TypeScript, kein Mount) |
| Composables | Vitest (Mock-Services via `vi.fn()`) |
| Components | Vitest + Vue Test Utils (`mount()`) |
| E2E (optional) | Playwright |

## Vitest-Konfiguration

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./tests/setup.ts'],
  }
})
```

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "noImplicitReturns": true,
    "noUnusedLocals": true
  }
}
```

## Model / Value Object Tests (kein Vue)

```typescript
// tests/unit/models/Email.spec.ts
import { describe, it, expect } from 'vitest'
import { Email } from '@/models/Email'

describe('Email', () => {
  it('normalisiert zu Kleinbuchstaben', () => {
    expect(Email.create('TEST@EXAMPLE.COM').toString()).toBe('test@example.com')
  })

  it('wirft bei ungültigem Format', () => {
    expect(() => Email.create('kein-at')).toThrow('Invalid email')
  })

  it('zwei gleiche Werte sind gleich', () => {
    const a = Email.create('max@test.de')
    const b = Email.create('max@test.de')
    expect(a.equals(b)).toBe(true)
  })
})
```

## Service Tests (MockApiClient)

```typescript
// tests/unit/services/UserService.spec.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { UserService } from '@/services/implementations/UserService'
import type { IApiClient } from '@/services/interfaces/IApiClient'

class MockApiClient implements IApiClient {
  private responses: Record<string, unknown> = {}

  setResponse(url: string, data: unknown) { this.responses[url] = data }

  async get<T>(url: string): Promise<T> { return this.responses[url] as T }
  async post<T>(_url: string, _data: unknown): Promise<T> { return this.responses['post'] as T }
  async delete(_url: string): Promise<void> {}
}

describe('UserService', () => {
  let api: MockApiClient
  let service: UserService

  beforeEach(() => {
    api = new MockApiClient()
    service = new UserService(api)
  })

  it('gibt alle User zurück', async () => {
    api.setResponse('/users', [{ id: '1', name: 'Max', email: 'max@test.de', role: 'user' }])
    const users = await service.getAll()
    expect(users).toHaveLength(1)
    expect(users[0].name).toBe('Max')
  })
})
```

## Composable Tests

```typescript
// tests/unit/composables/useUsers.spec.ts
import { describe, it, expect, vi } from 'vitest'
import { useUsers } from '@/composables/useUsers'
import type { IUserService } from '@/services/interfaces/IUserService'

const mockUser = { id: { toString: () => '1' }, name: 'Max', isAdmin: () => false }

describe('useUsers', () => {
  it('startet mit leerem State', () => {
    const service = { getAll: vi.fn(), create: vi.fn(), delete: vi.fn() } as unknown as IUserService
    const { users, loading, error } = useUsers(service)
    expect(users.value).toHaveLength(0)
    expect(loading.value).toBe(false)
    expect(error.value).toBeNull()
  })

  it('lädt User und setzt loading korrekt', async () => {
    const service = { getAll: vi.fn().mockResolvedValue([mockUser]) } as unknown as IUserService
    const { users, loading, load } = useUsers(service)

    const promise = load()
    expect(loading.value).toBe(true)
    await promise
    expect(loading.value).toBe(false)
    expect(users.value).toHaveLength(1)
  })

  it('setzt error bei Ladefehler', async () => {
    const service = { getAll: vi.fn().mockRejectedValue(new Error('Netzwerkfehler')) } as unknown as IUserService
    const { error, load } = useUsers(service)
    await load()
    expect(error.value).toBe('Netzwerkfehler')
  })
})
```

## Component Tests

```typescript
// tests/unit/components/UserList.spec.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import UserList from '@/components/UserList.vue'

const mockUser = { id: '1', name: 'Max', email: 'max@test.de' }

describe('UserList', () => {
  it('rendert einen Eintrag pro User', () => {
    const wrapper = mount(UserList, {
      props: { users: [mockUser, { ...mockUser, id: '2', name: 'Moritz' }] }
    })
    expect(wrapper.findAll('[data-testid="user-item"]')).toHaveLength(2)
    expect(wrapper.text()).toContain('Max')
  })

  it('zeigt Empty State wenn keine User', () => {
    const wrapper = mount(UserList, { props: { users: [] } })
    expect(wrapper.find('[data-testid="empty-state"]').exists()).toBe(true)
  })

  it('emittiert delete-Event mit User-ID', async () => {
    const wrapper = mount(UserList, { props: { users: [mockUser] } })
    await wrapper.find('[data-testid="delete-btn"]').trigger('click')
    expect(wrapper.emitted('delete')?.[0]).toEqual(['1'])
  })
})
```

## TDD-Regel für Vue

**TypeScript-Compilerfehler zählt als RED.** Ein nicht existierendes Interface oder fehlende Methode ist genauso ein gültiger RED-Schritt wie ein Laufzeitfehler.

```typescript
// 🔴 RED — dieser Import schlägt fehl, weil die Datei noch nicht existiert
import { UserService } from '@/services/implementations/UserService'  // ImportError = RED ✅
```
