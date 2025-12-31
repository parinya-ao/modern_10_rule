# example

### 1. Simple Flow & Async Clarity

**Principle:** Avoid nested callbacks. Use language features that make code read top-to-bottom.

**TypeScript**
Use `async/await` to flatten "callback hell" into a linear structure.

```typescript
// ❌ Non-Compliant (Callback Hell)
function processUser(id: string) {
    getUser(id)
        .then(user => {
            getPermissions(user.role)
                .then(perms => {
                    saveLog(user, perms).catch(err => console.error(err));
                });
        })
        .catch(err => console.error(err));
}

// ✅ Compliant (Linear Flow)
async function processUser(id: string): Promise<void> {
    try {
        const user = await getUser(id);
        const perms = await getPermissions(user.role);
        await saveLog(user, perms);
    } catch (error) {
        // Stack trace is preserved, and logic is readable
        console.error('Failed to process user', error);
    }
}

```

**Go**
Use `select` for clean concurrency control instead of complex channel logic.

```go
// ✅ Compliant
func processWithTimeout(ch <-chan int) {
    select {
    case val := <-ch:
        fmt.Println("Received:", val)
    case <-time.After(2 * time.Second):
        fmt.Println("Timeout: Operation took too long")
    }
}

```

---

### 2. Timeouts & Rate Limits

**Principle:** Never wait forever. Always assume the external system might hang.

**Go**
Use `context.Context` to pass deadlines down the call stack.

```go
// ✅ Compliant
func fetchData(ctx context.Context, url string) error {
    // Create a child context that dies after 5 seconds
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    // Pass context to the request
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return fmt.Errorf("request failed: %w", err)
    }
    defer resp.Body.Close()
    return nil
}

```

**Python**
Always set the `timeout` parameter in HTTP clients.

```python
# ✅ Compliant
import requests

def get_external_data(url):
    try:
        # Enforce a 5-second timeout. Without this, it could hang indefinitely.
        response = requests.get(url, timeout=5)
        response.raise_for_status()
        return response.json()
    except requests.Timeout:
        logger.error("Request timed out")
        return None

```

---

### 3. Strict Resource Management

**Principle:** Clean up resources immediately after use (RAII).

**Python**
Use `with` statements (Context Managers) to ensure file handles are closed even if errors occur.

```python
# ❌ Non-Compliant
f = open("data.txt")
data = f.read()
# If read() fails, f.close() is never reached
f.close()

# ✅ Compliant
with open("data.txt", "r") as f:
    data = f.read()
# File is automatically closed here

```

**Go**
Use `defer` immediately after resource allocation.

```go
// ✅ Compliant
f, err := os.Open("data.txt")
if err != nil {
    return err
}
defer f.Close() // Cleanup is guaranteed even if a panic occurs later

// ... process file ...

```

---

### 4. Small Functions & Single Responsibility

**Principle:** Functions should do one thing well and be visually short.

**Python**
Refactor complex logic into helper functions.

```python
# ✅ Compliant
def process_order(order):
    # This function orchestrates the flow but delegates the work
    validate_inventory(order)

    if charge_card(order.payment_details):
        ship_items(order)
        send_receipt(order.user)
    else:
        handle_payment_failure(order)

def validate_inventory(order):
    # Specific logic for checking stock...
    pass

```

---

### 5. Validate at the Edge

**Principle:** Use schemas to validate input before it enters your business logic.

**TypeScript**
Use `Zod` to ensure runtime data matches compile-time types.

```typescript
import { z } from "zod";

const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  age: z.number().min(18)
});

// ✅ Compliant
function handleRequest(input: unknown) {
  const result = UserSchema.safeParse(input);

  if (!result.success) {
    // Fail fast with a clear error
    throw new Error(`Validation failed: ${result.error}`);
  }

  // 'user' is now guaranteed to be valid and typed correctly
  const user = result.data;
  saveUser(user);
}

```

**Python**
Use `Pydantic` for data validation.

```python
from pydantic import BaseModel, EmailStr

class User(BaseModel):
    id: int
    email: EmailStr
    is_active: bool

# ✅ Compliant
def create_user(payload: dict):
    # Automatically validates types and format. Raises error if invalid.
    user = User(**payload)
    db.save(user)

```

---

### 6. No Global Mutable State

**Principle:** Avoid global variables; pass dependencies explicitly.

**Go**
Use Struct-based dependency injection instead of global `var`.

```go
// ❌ Non-Compliant
var db *sql.DB // Global variable accessible by everyone

// ✅ Compliant
type Server struct {
    db *sql.DB // Encapsulated state
}

func NewServer(db *sql.DB) *Server {
    return &Server{db: db}
}

func (s *Server) GetUser(id string) {
    // Access via the receiver method, clear dependency
    s.db.Query(...)
}

```

---

### 7. Handle Errors Explicitly

**Principle:** Never ignore an error. Wrap it with context if you pass it up.

**Go**
Check every error and provide context.

```go
// ✅ Compliant
func loadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        // Wrap the error: "failed to read config: open config.json: file not found"
        return nil, fmt.Errorf("failed to read config: %w", err)
    }
    return parseConfig(data), nil
}

```

**Python**
Catch specific exceptions, never bare `except:`.

```python
# ✅ Compliant
try:
    result = hazardous_operation()
except ValueError as e:
    logger.error(f"Invalid input: {e}")
    raise
except ConnectionError as e:
    logger.error(f"Network error: {e}")
    # Handle retry logic...

```

---

### 8. Avoid "Magic" Code

**Principle:** Code should be explicit and easy to trace by IDEs.

**Python**
Avoid dynamic attribute access (`__getattr__`) which breaks autocomplete.

```python
# ❌ Non-Compliant (Magic)
class MagicData:
    def __getattr__(self, name):
        # Where does this come from? Hard to trace.
        return self.data.get(name)

# ✅ Compliant (Explicit)
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age

```

---

### 9. Immutability First

**Principle:** Prefer creating new data over changing old data.

**JavaScript / TypeScript**
Use array methods that return new arrays.

```javascript
const items = [1, 2, 3];

// ❌ Non-Compliant (Mutates original array)
items.push(4);

// ✅ Compliant (Creates new array, original is safe)
const newItems = [...items, 4];
const doubled = items.map(n => n * 2);

```

---

### 10. Linting & Static Analysis

**Principle:** Automate enforcement.

**Configuration Example (`pyproject.toml` for Python)**
Enforce strict rules automatically.

```toml
[tool.ruff]
line-length = 88
select = [
    "E",   # pycodestyle errors
    "F",   # pyflakes
    "B",   # bugbear (finds common bugs)
    "UP",  # pyupgrade (modern syntax)
]

[tool.mypy]
strict = true
disallow_untyped_defs = true # Force type hints on everything

```
