# Go Service as an HTTP Client: Notes

## 1. Core idea
- A Go service can be a **server** (receiving requests) and a **client** (sending requests) at the same time.
- In microservices, when Service A calls Service B, **A acts as an HTTP client** and B acts as the server.
- Both roles use the same standard library: **`net/http`**.
  - Server side: `http.Handler`, `http.ResponseWriter`, `*http.Request`
  - Client side: `http.Client`, `http.NewRequest`, `client.Do`

## 2. The basic steps
1. **Prepare the payload**: convert a Go struct to JSON with `json.Marshal`.
2. **Build the request**: `http.NewRequestWithContext(ctx, method, url, body)`.
3. **Set headers**: for example `Content-Type: application/json`.
4. **Send it**: `client.Do(req)`.
5. **Close the body**: `defer resp.Body.Close()` (always).
6. **Check the status code**: `resp.StatusCode`.
7. **Decode the response**: `json.NewDecoder(resp.Body).Decode(&result)`.

## 3. Full example

```go
type AuthPayload struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}

type AuthResponse struct {
    Error   bool   `json:"error"`
    Message string `json:"message"`
    Data    any    `json:"data,omitempty"`
}

// Create ONE client and reuse it (it keeps a pool of connections)
var httpClient = &http.Client{Timeout: 5 * time.Second}

func callAuthService(ctx context.Context, p AuthPayload) (*AuthResponse, error) {
    // 1. Struct -> JSON
    body, err := json.Marshal(p)
    if err != nil {
        return nil, err
    }

    // 2. Build request (with context for cancellation)
    req, err := http.NewRequestWithContext(ctx, http.MethodPost,
        "http://authentication-service/authenticate", bytes.NewBuffer(body))
    if err != nil {
        return nil, err
    }

    // 3. Headers
    req.Header.Set("Content-Type", "application/json")

    // 4. Send
    resp, err := httpClient.Do(req)
    if err != nil {
        return nil, err // network error, timeout, DNS failure, etc.
    }
    // 5. Always close the body
    defer resp.Body.Close()

    // 6. Check status code
    if resp.StatusCode != http.StatusAccepted {
        return nil, fmt.Errorf("auth service returned status %d", resp.StatusCode)
    }

    // 7. JSON -> struct
    var result AuthResponse
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, err
    }
    return &result, nil
}
```

Using it inside a handler (the service is a server and a client at the same time):

```go
func (app *Config) authenticate(w http.ResponseWriter, r *http.Request) {
    var p AuthPayload
    // ... read p from the incoming request ...

    result, err := callAuthService(r.Context(), p) // pass the incoming request's context
    if err != nil {
        app.errorJSON(w, err)
        return
    }
    app.writeJSON(w, http.StatusAccepted, result)
}
```

## 4. Key points to remember
- **Always set a timeout.** `&http.Client{}` has none, so a slow service can make your service hang forever.
- **Reuse the client.** Don't create a new `http.Client` per request; it is safe for concurrent use.
- **Always `defer resp.Body.Close()`** right after checking `err`, or connections leak.
- **`err == nil` does NOT mean success.** A 404 or 500 still returns `err == nil`, so you must check `resp.StatusCode` yourself.
- **Pass the context** (`r.Context()`). If the original user request is canceled, the downstream call is canceled too.
- **Don't ignore errors** from `json.Marshal` (avoid `jsonData, _ := ...`).
- **Don't hardcode URLs.** Read service addresses from environment variables or config.
- `json.MarshalIndent` is only for human-readable output; use `json.Marshal` between services.

## 5. Service names as hostnames
- In `http://authentication-service/authenticate`, `authentication-service` is **not** a public domain.
- It is the **service name in Docker Compose / Kubernetes**. Their internal DNS resolves it to the container's IP.
- This works only for services inside the same Docker network or cluster.

## 6. Shortcut helpers (simple cases)
```go
resp, err := http.Get(url)
resp, err := http.Post(url, "application/json", bytes.NewBuffer(body))
```
- These are quick, but they use `http.DefaultClient`, which has **no timeout**. Prefer your own client in production.

## 7. Other ways services communicate
| Method | When to use |
|---|---|
| **HTTP + JSON** | Simple and universal; the default choice |
| **gRPC** | Fast, typed contracts (protobuf); internal service-to-service calls |
| **Message queue** (RabbitMQ, Kafka) | Async; fire-and-forget (logs, emails, events) |
| **net/rpc** | Go-to-Go only; rarely used today |

## 8. One-line summary
> A Go service becomes an HTTP client by building a request with `http.NewRequestWithContext`, sending it with a reusable `http.Client` that has a timeout, then always closing the body, checking the status code, and decoding the JSON response.