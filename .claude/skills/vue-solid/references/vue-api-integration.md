# API Integration — Vue 3 Patterns

## IApiClient Interface

```typescript
// services/interfaces/IApiClient.ts
export interface IApiClient {
  get<T>(url: string): Promise<T>
  post<T>(url: string, data: unknown): Promise<T>
  put<T>(url: string, data: unknown): Promise<T>
  delete(url: string): Promise<void>
}
```

Das Interface entkoppelt Services von Axios — Services wissen nicht, welches HTTP-Framework verwendet wird.

## AxiosApiClient Implementation

```typescript
// services/implementations/AxiosApiClient.ts
import axios, { type AxiosInstance } from 'axios'
import type { IApiClient } from '../interfaces/IApiClient'

export class AxiosApiClient implements IApiClient {
  private readonly client: AxiosInstance

  constructor(baseURL: string) {
    this.client = axios.create({
      baseURL,
      headers: { 'Content-Type': 'application/json' },
    })

    // Response-Interceptor: Fehler zentralisieren
    this.client.interceptors.response.use(
      response => response.data,
      error => {
        const message = error.response?.data?.detail ?? error.message
        throw new Error(message)
      }
    )
  }

  async get<T>(url: string): Promise<T> {
    return this.client.get(url)
  }

  async post<T>(url: string, data: unknown): Promise<T> {
    return this.client.post(url, data)
  }

  async put<T>(url: string, data: unknown): Promise<T> {
    return this.client.put(url, data)
  }

  async delete(url: string): Promise<void> {
    await this.client.delete(url)
  }
}
```

## SSE (Server-Sent Events)

Für Echtzeit-Streams (z.B. ipNINX Dashboard):

```typescript
// services/implementations/EventStreamService.ts
export class EventStreamService implements IEventStreamService {
  private eventSource: EventSource | null = null

  connect(url: string, onEvent: (data: unknown) => void): void {
    this.eventSource = new EventSource(url)
    this.eventSource.onmessage = (event) => {
      onEvent(JSON.parse(event.data))
    }
    this.eventSource.onerror = () => {
      this.disconnect()
    }
  }

  disconnect(): void {
    this.eventSource?.close()
    this.eventSource = null
  }
}

// Composable für SSE
export function useEventStream(service: IEventStreamService, url: string) {
  const events = ref<unknown[]>([])
  const connected = ref(false)

  function start() {
    service.connect(url, (data) => events.value.push(data))
    connected.value = true
  }

  onUnmounted(() => service.disconnect())

  return { events, connected, start }
}
```

## Fehlerbehandlung

```typescript
// Zentral im AxiosApiClient (Interceptor) — nicht in jedem Service wiederholen

// Im Service: Domain-spezifische Fehler werfen
async create(data: CreateUserDto): Promise<User> {
  try {
    const dto = await this.api.post<UserDto>('/users', data)
    return User.fromDto(dto)
  } catch (e) {
    if (e instanceof Error && e.message.includes('already exists')) {
      throw new Error('E-Mail bereits vergeben')  // Domain-Fehler
    }
    throw e  // Andere Fehler weiterwerfen
  }
}

// Im Composable: Fehler in ref speichern
async function create(data: CreateUserDto) {
  try {
    const user = await userService.create(data)
    users.value.push(user)
  } catch (e) {
    error.value = e instanceof Error ? e.message : 'Fehler beim Erstellen'
  }
}
```

## Auth-Token (Axios Interceptor)

```typescript
// services/implementations/AxiosApiClient.ts
constructor(baseURL: string, private readonly getToken: () => string | null) {
  this.client = axios.create({ baseURL })

  // Request-Interceptor: Token anhängen
  this.client.interceptors.request.use(config => {
    const token = getToken()
    if (token) config.headers.Authorization = `Bearer ${token}`
    return config
  })
}

// main.ts
const apiClient = new AxiosApiClient(
  import.meta.env.VITE_API_URL,
  () => localStorage.getItem('token')
)
```

## MockApiClient für Tests

```typescript
// tests/mocks/MockApiClient.ts
import type { IApiClient } from '@/services/interfaces/IApiClient'

export class MockApiClient implements IApiClient {
  private responses = new Map<string, unknown>()
  public calls: Array<{ method: string; url: string; data?: unknown }> = []

  setResponse(url: string, data: unknown) {
    this.responses.set(url, data)
  }

  async get<T>(url: string): Promise<T> {
    this.calls.push({ method: 'GET', url })
    if (!this.responses.has(url)) throw new Error(`No mock for GET ${url}`)
    return this.responses.get(url) as T
  }

  async post<T>(url: string, data: unknown): Promise<T> {
    this.calls.push({ method: 'POST', url, data })
    return this.responses.get(`POST:${url}`) as T
  }

  async put<T>(url: string, data: unknown): Promise<T> {
    this.calls.push({ method: 'PUT', url, data })
    return this.responses.get(`PUT:${url}`) as T
  }

  async delete(url: string): Promise<void> {
    this.calls.push({ method: 'DELETE', url })
  }
}
```
