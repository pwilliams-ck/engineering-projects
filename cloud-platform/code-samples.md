# Code Samples

Anonymized patterns from production Go services.

---

## Table of Contents

- [Graceful Shutdown](#graceful-shutdown)
- [Retry with Exponential Backoff](#retry-with-exponential-backoff)
- [Functional Options Pattern](#functional-options-pattern)
- [HTTP Client Configuration](#http-client-configuration)
- [Middleware Patterns](#middleware-patterns)
- [Error Handling](#error-handling)
- [Webhook Signature Validation](#webhook-signature-validation)
- [Input Validation](#input-validation)

---

## Graceful Shutdown

Signal handling with proper cleanup and timeout.

```go
func (app *application) serve() error {
    srv := &http.Server{
        Addr:         fmt.Sprintf(":%d", app.config.port),
        Handler:      app.routes(),
        IdleTimeout:  time.Minute,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        ErrorLog:     slog.NewLogLogger(app.logger.Handler(), slog.LevelError),
    }

    shutdownError := make(chan error)

    go func() {
        quit := make(chan os.Signal, 1)
        signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

        s := <-quit
        app.logger.Info("shutting down server...", "signal", s.String())

        ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
        defer cancel()

        shutdownError <- srv.Shutdown(ctx)
    }()

    app.logger.Info("starting server", "addr", srv.Addr, "env", app.config.env)

    err := srv.ListenAndServe()
    if !errors.Is(err, http.ErrServerClosed) {
        return err
    }

    err = <-shutdownError
    if err != nil {
        return err
    }

    app.logger.Info("stopped server", "addr", srv.Addr)
    return nil
}
```

**Key points:**
- Channel-based coordination between main goroutine and signal handler
- Context with timeout prevents hanging on slow connections
- Differentiates between normal shutdown (`http.ErrServerClosed`) and errors

---

## Retry with Exponential Backoff

Production retry logic with jitter and context cancellation.

```go
type retryableHTTPClient struct {
    client      *http.Client
    maxRetries  int
    retryDelay  time.Duration
    maxJitter   time.Duration
    retryPolicy RetryPolicy
}

type RetryPolicy func(*http.Response, error) bool

func DefaultRetryPolicy(resp *http.Response, err error) bool {
    if err != nil {
        return true
    }
    return resp.StatusCode == http.StatusBadRequest || resp.StatusCode >= 500
}

func (c *retryableHTTPClient) DoWithRetry(req *http.Request) (*http.Response, error) {
    var resp *http.Response
    var err error
    var body []byte

    // Read body into memory for retry reuse
    if req.Body != nil {
        if _, ok := req.Body.(io.Seeker); !ok {
            body, err = io.ReadAll(req.Body)
            if err != nil {
                return nil, fmt.Errorf("error reading request body: %w", err)
            }
            _ = req.Body.Close()
        }
    }

    for i := 0; i < c.maxRetries; i++ {
        // Reset body for each attempt
        if body != nil {
            req.Body = io.NopCloser(bytes.NewReader(body))
        }

        resp, err = c.client.Do(req)
        if !c.retryPolicy(resp, err) {
            return resp, err
        }

        if i == c.maxRetries-1 {
            break
        }

        // Exponential backoff with jitter
        sleepDuration := c.retryDelay * time.Duration(math.Pow(2, float64(i)))
        jitter := time.Duration(rand.Int63n(int64(c.maxJitter)))

        select {
        case <-req.Context().Done():
            return nil, req.Context().Err()
        case <-time.After(sleepDuration + jitter):
        }
    }

    return resp, err
}
```

**Key points:**
- Body buffering allows POST/PUT retry without re-reading
- Jitter prevents thundering herd on service recovery
- Context cancellation respected between retries
- Pluggable retry policy

---

## Functional Options Pattern

Clean, testable client initialization.

```go
type Client struct {
    httpClient *retryableHTTPClient
    BaseURL    string
    APIKey     string
}

type ClientOption func(*Client)

func NewClient(options ...ClientOption) *Client {
    client := &Client{
        BaseURL: os.Getenv("SERVICE_URL"),
        APIKey:  os.Getenv("SERVICE_API_KEY"),
        httpClient: &retryableHTTPClient{
            client:      createHTTPClient(),
            maxRetries:  3,
            retryDelay:  time.Second,
            maxJitter:   time.Second,
            retryPolicy: DefaultRetryPolicy,
        },
    }

    for _, option := range options {
        option(client)
    }

    return client
}

// Option functions
func WithBaseURL(url string) ClientOption {
    return func(c *Client) {
        c.BaseURL = url
    }
}

func WithRetryPolicy(maxRetries int, delay, jitter time.Duration, policy RetryPolicy) ClientOption {
    return func(c *Client) {
        c.httpClient.maxRetries = maxRetries
        c.httpClient.retryDelay = delay
        c.httpClient.maxJitter = jitter
        c.httpClient.retryPolicy = policy
    }
}
```

**Usage:**
```go
// Production: uses environment defaults
client := NewClient()

// Testing: override for deterministic behavior
client := NewClient(
    WithBaseURL("http://localhost:8080"),
    WithRetryPolicy(1, 0, 0, func(*http.Response, error) bool { return false }),
)
```

---

## HTTP Client Configuration

Production-ready transport settings.

```go
func createHTTPClient() *http.Client {
    return &http.Client{
        Timeout: 30 * time.Second,
        Transport: &http.Transport{
            TLSClientConfig: &tls.Config{
                MinVersion: tls.VersionTLS12,
            },
            DialContext: (&net.Dialer{
                Timeout:   30 * time.Second,
                KeepAlive: 30 * time.Second,
            }).DialContext,
            ForceAttemptHTTP2:     true,
            MaxIdleConns:          100,
            MaxConnsPerHost:       100,
            MaxIdleConnsPerHost:   100,
            IdleConnTimeout:       90 * time.Second,
            TLSHandshakeTimeout:   10 * time.Second,
            ExpectContinueTimeout: 1 * time.Second,
        },
    }
}
```

**Key points:**
- TLS 1.2 minimum (security baseline)
- Connection pooling prevents socket exhaustion
- HTTP/2 enabled for multiplexing
- Explicit timeouts at every layer

---

## Middleware Patterns

### Panic Recovery

```go
func (app *application) gracefulRecovery(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                w.Header().Set("Connection", "close")
                app.serverErrorResponse(w, r, fmt.Errorf("%s", err))
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

### Request Logging

```go
func (app *application) logRequest(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        app.logger.Info("request",
            "ip", r.RemoteAddr,
            "proto", r.Proto,
            "method", r.Method,
            "uri", r.URL.RequestURI(),
        )
        next.ServeHTTP(w, r)
    })
}
```

### API Key Authentication

```go
func (app *application) requireAPIKey(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        apiKey := os.Getenv("API_KEY")

        // Skip auth if not configured (development mode)
        if apiKey == "" {
            next.ServeHTTP(w, r)
            return
        }

        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            app.unauthorizedResponse(w, r, "missing Authorization header")
            return
        }

        // Accept "Bearer <key>" or "Api-Key <key>"
        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 {
            app.unauthorizedResponse(w, r, "invalid Authorization format")
            return
        }

        scheme := strings.ToLower(parts[0])
        if scheme != "bearer" && scheme != "api-key" {
            app.unauthorizedResponse(w, r, "use 'Bearer' or 'Api-Key' scheme")
            return
        }

        if parts[1] != apiKey {
            app.unauthorizedResponse(w, r, "invalid API key")
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

---

## Error Handling

Consistent JSON error responses with proper logging.

```go
type envelope map[string]any

func (app *application) logError(r *http.Request, err error) {
    app.logger.Error(err.Error(),
        "method", r.Method,
        "uri", r.URL.RequestURI(),
    )
}

func (app *application) errorResponse(w http.ResponseWriter, r *http.Request, status int, message any) {
    env := envelope{"error": message}

    err := app.writeJSON(w, status, env, nil)
    if err != nil {
        app.logError(r, err)
        w.WriteHeader(http.StatusInternalServerError)
    }
}

func (app *application) serverErrorResponse(w http.ResponseWriter, r *http.Request, err error) {
    app.logError(r, err)
    app.errorResponse(w, r, http.StatusInternalServerError,
        "the server encountered a problem and could not process your request")
}

func (app *application) badRequestResponse(w http.ResponseWriter, r *http.Request, err error) {
    app.errorResponse(w, r, http.StatusBadRequest, err.Error())
}

func (app *application) failedValidationResponse(w http.ResponseWriter, r *http.Request, errors map[string]string) {
    app.errorResponse(w, r, http.StatusUnprocessableEntity, errors)
}

func (app *application) unauthorizedResponse(w http.ResponseWriter, r *http.Request, message string) {
    app.errorResponse(w, r, http.StatusUnauthorized, message)
}
```

**Key points:**
- Generic `errorResponse` handles all cases
- Fallback to raw 500 if JSON writing fails
- User-facing messages don't leak internal details
- Structured logging with request context

---

## Webhook Signature Validation

HMAC-SHA256 validation with replay attack prevention.

```go
func ValidateWebhookSignature(signature, timestamp, secret string, body []byte, maxAge int64) error {
    if signature == "" || timestamp == "" {
        return fmt.Errorf("missing required headers")
    }

    // Compute expected signature: HMAC-SHA256(secret, timestamp + body)
    payload := timestamp + string(body)
    h := hmac.New(sha256.New, []byte(secret))
    h.Write([]byte(payload))
    expected := hex.EncodeToString(h.Sum(nil))

    if expected != signature {
        return fmt.Errorf("invalid signature")
    }

    // Verify timestamp to prevent replay attacks
    ts, err := strconv.ParseInt(timestamp, 10, 64)
    if err != nil {
        return fmt.Errorf("invalid timestamp format: %w", err)
    }

    if time.Now().Unix()-ts > maxAge {
        return fmt.Errorf("timestamp older than %d seconds", maxAge)
    }

    return nil
}
```

**Key points:**
- Timestamp in signature prevents replay attacks
- `maxAge` configurable per-endpoint sensitivity
- Constant-time comparison would be better (use `hmac.Equal` in production)

---

## Input Validation

Fluent validation API with generics.

```go
var EmailRX = regexp.MustCompile(
    `^[a-zA-Z0-9.!#$%&'*+/=?^_` + "`" + `{|}~-]+@[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*$`,
)

type Validator struct {
    Errors map[string]string
}

func New() *Validator {
    return &Validator{Errors: make(map[string]string)}
}

func (v *Validator) Valid() bool {
    return len(v.Errors) == 0
}

func (v *Validator) AddError(key, message string) {
    if _, exists := v.Errors[key]; !exists {
        v.Errors[key] = message
    }
}

func (v *Validator) Check(ok bool, key, message string) {
    if !ok {
        v.AddError(key, message)
    }
}

// Generic helpers
func PermittedValue[T comparable](value T, permitted ...T) bool {
    return slices.Contains(permitted, value)
}

func Matches(value string, rx *regexp.Regexp) bool {
    return rx.MatchString(value)
}

func Unique[T comparable](values []T) bool {
    seen := make(map[T]bool)
    for _, v := range values {
        seen[v] = true
    }
    return len(values) == len(seen)
}
```

**Usage:**
```go
func (app *application) handleCreateUser(w http.ResponseWriter, r *http.Request) {
    var input struct {
        Email     string `json:"email"`
        FirstName string `json:"first_name"`
        Role      string `json:"role"`
    }

    if err := app.readJSON(w, r, &input); err != nil {
        app.badRequestResponse(w, r, err)
        return
    }

    v := validator.New()
    v.Check(input.Email != "", "email", "must be provided")
    v.Check(Matches(input.Email, EmailRX), "email", "must be valid")
    v.Check(input.FirstName != "", "first_name", "must be provided")
    v.Check(PermittedValue(input.Role, "admin", "user", "viewer"), "role", "must be admin, user, or viewer")

    if !v.Valid() {
        app.failedValidationResponse(w, r, v.Errors)
        return
    }

    // ... create user
}
```

**Response on validation failure:**
```json
{
    "error": {
        "email": "must be valid",
        "role": "must be admin, user, or viewer"
    }
}
```
