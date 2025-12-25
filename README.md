# Standard Operating Procedure: The Modern Power of 10 for Software Engineering Reliability

## บทนำ

เอกสารฉบับนี้กำหนดมาตรฐานการปฏิบัติงาน (Standard Operating Procedure - SOP) สำหรับกระบวนการพัฒนาซอฟต์แวร์ โดยอ้างอิงและปรับปรุงจากหลักการ "NASA's Power of 10 Rules for Safety-Critical Code" เพื่อให้สอดคล้องกับสภาพแวดล้อมการพัฒนาสมัยใหม่ (Modern Development Stacks: Python, TypeScript, Go)

**วัตถุประสงค์:** เพื่อสร้างความมั่นใจในคุณภาพ ความน่าเชื่อถือ (Reliability) และความสามารถในการดูแลรักษา (Maintainability) ของระบบซอฟต์แวร์ โดยเฉพาะอย่างยิ่งในองค์กรที่มีอัตราการหมุนเวียนของบุคลากรสูง (High Turnover) มาตรฐานเหล่านี้ถูกออกแบบมาเพื่อลดหนี้ทางเทคนิค (Technical Debt) และลดภาระทางปัญญา (Cognitive Load) ให้กับนักพัฒนาใหม่ที่ต้องเข้ามาสานต่องาน

---

## 1. Simple Flow & Async Clarity (ความชัดเจนและเรียบง่ายของลำดับการทำงาน)

**หลักการ:** โครงสร้างการควบคุม (Control Flow) ของโปรแกรมจะต้องมีความเป็นเส้นตรง (Linearity) มากที่สุดเท่าที่จะเป็นไปได้ เพื่อให้สามารถอ่านทำความเข้าใจได้ในการกวาดสายตาครั้งเดียว (Single-pass readability) โดยปราศจากการกระโดดข้ามไปมาของบริบท (Context Switching)

### รายละเอียดและแนวทางปฏิบัติ

ในระบบสมัยใหม่ ความซับซ้อนมักเกิดจากการทำงานแบบ Asynchronous การจัดการ State และ Concurrency ปัญหาที่พบบ่อยคือ Callback Hell หรือการใช้ Goroutine/Thread ที่ซับซ้อนจนไม่สามารถติดตามสถานะการทำงานได้

1. **การขจัด Callback Hell:** ห้ามใช้ Nested Callbacks เกิน 2 ระดับในทุกกรณี ให้ใช้โครงสร้างภาษาที่ช่วยให้การเขียนโค้ดดูเป็นลำดับ (Sequential-looking code)
2. **Explicit Data Flow:** การส่งข้อมูลระหว่าง Thread หรือ Goroutine ต้องมีทิศทางที่ชัดเจน (Unidirectional Data Flow) หลีกเลี่ยงการแชร์ State ที่ซับซ้อน

### Implementation Guidelines

**TypeScript:**
ใช้ `async/await` แทน `Promise.then().catch()` เพื่อให้โครงสร้างของ Block `try/catch` ครอบคลุม Logic ทั้งหมด การใช้ Promise Chains ที่ยาวเกินไปทำให้ Call Stack ไม่ชัดเจนเมื่อเกิดข้อผิดพลาด

**Code Example:**

```typescript
// Non-Compliant (Complex Flow)
function processData() {
    getData()
        .then(data => transform(data))
        .then(result => save(result))
        .catch(err => handleError(err)); // Error context is lost
}

// Compliant (Linear Flow)
async function processData(): Promise<void> {
    try {
        const data = await getData();
        const result = await transform(data);
        await save(result);
    } catch (error) {
        // Stack trace preserved clearly
        handleError(new Error('Failed to process data', { cause: error }));
    }
}

```

**Go:**
หลีกเลี่ยงการสร้าง Channel ที่ส่งข้อมูลไป-กลับแบบไร้ทิศทาง ให้ใช้ Pattern เช่น Pipeline หรือ Fan-out/Fan-in ที่มีการควบคุมชัดเจน และใช้ `select` statement ในการจัดการ blocking operations

---

## 2. Timeouts & Rate Limits (การกำหนดขอบเขตเวลาและการทำงาน)

**หลักการ:** ระบบต้องมีคุณสมบัติ Deterministic Latency ทุกการทำงานที่มีการรอคอย (Blocking Operations) ต้องมีจุดสิ้นสุดที่แน่นอน ห้ามปล่อยให้ Process รอคอยทรัพยากรอย่างไม่มีกำหนด (Indefinite Waiting)

### รายละเอียดและแนวทางปฏิบัติ

การรอคอย I/O (Network, Disk, Database) โดยไม่มี Timeout เป็นสาเหตุหลักของระบบค้าง (Hang) และ Resource Exhaustion เมื่อระบบปลายทางมีปัญหา

1. **Mandatory Timeouts:** การเรียก Network Request, Database Query หรือการรอ Lock จะต้องมีการกำหนด Timeout เสมอ
2. **Circuit Breaker Pattern:** เมื่อระบบภายนอกล้มเหลวซ้ำๆ ระบบต้องตัดการเชื่อมต่อทันที (Fail Fast) เพื่อป้องกันการสะสมของ Request ที่รอคอย

### Implementation Guidelines

**Go:**
ใช้ `context.Context` ในการส่ง Deadline ไปยังทุก Function ที่มีการทำ I/O

**Code Example:**

```go
// Compliant
func fetchData(ctx context.Context, url string) ([]byte, error) {
    // สร้าง Child Context พร้อม Timeout 2 วินาที
    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        // หากเกินเวลา err จะเป็น context.DeadlineExceeded
        return nil, fmt.Errorf("request failed: %w", err)
    }
    defer resp.Body.Close()

    return io.ReadAll(resp.Body)
}

```

---

## 3. Strict Resource Management (การบริหารจัดการทรัพยากรอย่างเคร่งครัด)

**หลักการ:** Resource Acquisition Is Initialization (RAII) แบบประยุกต์ ผู้ที่ร้องขอทรัพยากร (Memory, File Descriptor, Socket, Database Connection) มีหน้าที่รับผิดชอบในการคืนทรัพยากรนั้นทันทีที่เสร็จสิ้นภารกิจ

### รายละเอียดและแนวทางปฏิบัติ

Garbage Collector (GC) จัดการเพียง Memory ใน Heap แต่ไม่สามารถจัดการ System Resources อื่นๆ ได้ การละเลยจุดนี้จะนำไปสู่ Memory Leak และ File Descriptor Exhaustion

1. **Scope-based Cleanup:** ทรัพยากรต้องถูกคืนทันทีที่หลุดออกจาก Scope การทำงาน
2. **Explicit Cleanup in UI:** สำหรับ Frontend Frameworks การผูก Event Listener หรือ Subscription ต้องมีการ Unsubscribe ในขั้นตอน Teardown ของ Component เสมอ

### Implementation Guidelines

**Python:**
บังคับใช้ Context Managers (`with` statement) สำหรับการเปิดไฟล์หรือ Network connections ห้ามเรียก `.close()` เองด้วยมือเพราะอาจเกิด Error ก่อนถึงบรรทัดนั้น

**Code Example:**

```python
# Compliant
def process_log(file_path: str):
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            for line in f:
                process_line(line)
    except IOError as e:
        logger.error(f"Failed to process log: {e}")
    # File closed automatically here

```

**Go:**
ใช้ `defer` ทันทีหลังจากได้รับ resource เพื่อประกันการทำงานแม้จะเกิด Panic

```go
// Compliant
mu.Lock()
defer mu.Unlock() // ปลดล็อคเสมอ

```

---

## 4. Small Functions & Single Responsibility (ฟังก์ชันขนาดเล็กและหน้าที่เดียว)

**หลักการ:** ลดความซับซ้อน (Cyclomatic Complexity) โดยการยึดถือหลักการ Single Responsibility Principle (SRP) อย่างเคร่งครัด ฟังก์ชันหนึ่งต้องมีเหตุผลในการเปลี่ยนแปลงเพียงเหตุผลเดียว

### รายละเอียดและแนวทางปฏิบัติ

ฟังก์ชันที่มีขนาดใหญ่เกินไปทำให้ยากต่อการเขียน Unit Test และเพิ่ม Cognitive Load แก่นักพัฒนาใหม่ที่ต้องทำความเข้าใจ

1. **Visual Limit:** ความยาวของฟังก์ชันไม่ควรเกิน 1 หน้าจอ (ประมาณ 40-60 บรรทัด) โดยไม่รวม Comment
2. **Abstraction Level:** ทุกบรรทัดในฟังก์ชันควรอยู่ในระดับ Abstraction เดียวกัน ไม่ควรผสม Business Logic ระดับสูงเข้ากับ Low-level implementation details

### Implementation Guidelines

**Refactoring Strategy:**
หากจำเป็นต้องเขียน Comment เพื่ออธิบาย Section ต่างๆ ภายในฟังก์ชันเดียว แสดงว่าฟังก์ชันนั้นควรถูกแตกออกเป็นฟังก์ชันย่อย (Extract Method)

**Code Example (Concept):**

```python
# Compliant
def complete_purchase(order: Order, user: User):
    # High-level orchestration
    validate_inventory(order)
    payment_result = process_payment(user.payment_method, order.total)
    if payment_result.success:
        ship_items(order)
        send_receipt(user, order)
    else:
        handle_payment_failure(order)

```

---

## 5. Validate at the Edge (การตรวจสอบข้อมูล ณ จุดนำเข้า)

**หลักการ:** ใช้หลักการ Zero Trust กับข้อมูลภายนอก ข้อมูลที่เข้าสู่ระบบ (Input) ถือว่าเป็นข้อมูลที่ไม่ปลอดภัย (Tainted) จนกว่าจะผ่านกระบวนการตรวจสอบโครงสร้างและชนิดข้อมูล (Schema Validation)

### รายละเอียดและแนวทางปฏิบัติ

การตรวจสอบข้อมูลกระจัดกระจาย (Ad-hoc Validation) ภายใน Business Logic ทำให้โค้ดสกปรกและดูแลรักษายาก การตรวจสอบควรเกิดขึ้นที่ "ขอบ" (Edge) ของระบบ เช่น API Handler หรือ UI Form Submit

1. **Strict Schema:** ใช้ Library ในการกำหนด Schema และตรวจสอบข้อมูล แทนการเขียน if/else เช็คเอง
2. **Fail Fast:** หากข้อมูลผิดรูปแบบ ต้องปฏิเสธทันทีด้วย Error ที่ชัดเจน

### Implementation Guidelines

**TypeScript:**
ใช้ `Zod` หรือ `Yup` ในการทำ Runtime Validation เพื่อให้มั่นใจว่า TypeScript Type ตรงกับข้อมูลจริงขณะ Runtime

**Code Example:**

```typescript
import { z } from "zod";

const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  role: z.enum(["admin", "user"]),
});

type User = z.infer<typeof UserSchema>;

// Input data is 'unknown' until validated
function handleRequest(input: unknown) {
  const result = UserSchema.safeParse(input);
  if (!result.success) {
    throw new ValidationError(result.error);
  }
  const user: User = result.data; // Type-safe and validated
  service.createUser(user);
}

```

---

## 6. No Global Mutable State (การห้ามใช้สถานะร่วมที่เปลี่ยนแปลงได้)

**หลักการ:** หลีกเลี่ยง Shared Mutable State ซึ่งเป็นสาเหตุหลักของ Race Conditions และ Side Effects ที่คาดเดาไม่ได้ ทำให้การ Debug และ Test เป็นไปได้ยาก

### รายละเอียดและแนวทางปฏิบัติ

ตัวแปร Global ที่ใครก็สามารถแก้ไขค่าได้ ทำให้ระบบมีความเชื่อมโยงกันสูง (High Coupling) และซ่อน Dependencies ที่แท้จริง

1. **Encapsulation:** เก็บ State ไว้ภายใน Object หรือ Structure ที่เกี่ยวข้องเท่านั้น
2. **Dependency Injection (DI):** ส่ง Dependencies (เช่น Database config, User session) ผ่าน Parameter หรือ Constructor แทนการเข้าถึง Global Variable

### Implementation Guidelines

**Go:**
ห้ามใช้ `var` ประกาศ Global variable ในระดับ Package ยกเว้นค่าคงที่ (Const)

**Code Example:**

```go
// Non-Compliant
var db *sql.DB // Global mutable state

// Compliant
type Service struct {
    db *sql.DB // Encapsulated state
}

func NewService(db *sql.DB) *Service {
    return &Service{db: db}
}

func (s *Service) CreateUser(u User) error {
    return s.db.Exec(...) // Access via method receiver
}

```

---

## 7. Handle Errors Explicitly (การจัดการข้อผิดพลาดอย่างชัดเจน)

**หลักการ:** ข้อผิดพลาดต้องไม่ถูกละเลย (Silenced) ทุก Error ต้องได้รับการตรวจสอบ บันทึก (Log) หรือส่งต่อ (Propagate) อย่างเหมาะสม

### รายละเอียดและแนวทางปฏิบัติ

การใช้ Empty Catch Blocks (`try { ... } catch {}`) หรือการละเลย Return Value ที่เป็น Error ในภาษา Go คือการซ่อนปัญหาที่แท้จริง ทำให้เมื่อระบบล่ม ไม่สามารถหาสาเหตุได้

1. **No Silent Failures:** ห้ามเขียน Code ที่ "กลืน" Error โดยไม่มีการ Log
2. **Error Wrapping:** เมื่อส่งต่อ Error ให้เพิ่ม Context เข้าไปด้วยเสมอ (เช่น จาก "File not found" เป็น "Cannot load config: File not found")

### Implementation Guidelines

**Python:**
ห้ามใช้ `except:` (Bare except) ให้ระบุ Exception class เสมอ

```python
# Compliant
try:
    perform_risky_operation()
except ValueError as e:
    logger.warning(f"Invalid input: {e}")
    raise # Propagate error if functionality is critical
except Exception as e:
    logger.critical(f"System error: {e}")
    raise

```

**Go:**
ต้องตรวจสอบ `if err != nil` ทุกครั้ง ห้ามใช้ `_` เพื่อละเลย Error

---

## 8. Avoid "Magic" Code (การหลีกเลี่ยงโค้ดที่ซับซ้อนซ่อนเงื่อน)

**หลักการ:** Explicit is better than Implicit. โค้ดควรอ่านแล้วเข้าใจได้ว่ามันทำงานอย่างไรโดยไม่ต้องรู้กลไกเบื้องหลังที่ซับซ้อน (Metaprogramming)

### รายละเอียดและแนวทางปฏิบัติ

การใช้ Reflection, Decorators ที่ซับซ้อน, หรือ Magic Methods (`__getattr__`) ทำให้ IDE ไม่สามารถทำ Autocomplete หรือ Static Analysis ได้อย่างถูกต้อง และทำให้นักพัฒนาใหม่สับสน

1. **Avoid Reflection:** หลีกเลี่ยงการใช้ Reflection ใน Runtime path ปกติ ยกเว้นจำเป็นจริงๆ (เช่น เขียน Library ทั่วไป)
2. **Traceability:** โค้ดควรสามารถกด "Go to Definition" ใน IDE แล้วเจอจุดที่ทำงานจริงทันที

### Implementation Guidelines

เน้นการเขียนโค้ดแบบตรงไปตรงมา (Imperative) แม้จะต้องเขียนโค้ดซ้ำบ้าง (Boilerplate) แต่แลกมาด้วยความชัดเจนในการอ่านและการตรวจสอบ

---

## 9. Immutability First (การให้ความสำคัญกับข้อมูลที่ไม่เปลี่ยนแปลง)

**หลักการ:** ปฏิบัติต่อข้อมูลเสมือนว่าเป็นค่าคงที่ (Immutable) หากต้องการเปลี่ยนแปลงข้อมูล ให้สร้างสำเนาใหม่ (Copy) แทนการแก้ไขข้อมูลเดิม (Mutation)

### รายละเอียดและแนวทางปฏิบัติ

Mutation เป็นบ่อเกิดของ Bug ประเภท "Spooky action at a distance" (ส่วนหนึ่งของโค้ดไปแก้ค่าที่อีกส่วนกำลังใช้อยู่) และปัญหา Concurrency

1. **Prefer Const/Final:** ประกาศตัวแปรเป็นค่าคงที่ไว้ก่อนเสมอหากทำได้
2. **Pure Functions:** พยายามเขียนฟังก์ชันให้เป็น Pure Function (Output ขึ้นอยู่กับ Input เท่านั้น และไม่มี Side Effect)

### Implementation Guidelines

**JavaScript / React:**
ห้ามใช้ Array methods ที่เปลี่ยนแปลงค่าเดิม (เช่น `push`, `splice`) ให้ใช้ Method ที่ return ค่าใหม่ (เช่น `map`, `filter`, spread operator)

```javascript
// Compliant
const addItem = (items, newItem) => {
    return [...items, newItem]; // Returns new array
};

```

---

## 10. Linting & Static Analysis (การบังคับใช้กฎด้วยเครื่องมืออัตโนมัติ)

**หลักการ:** กฎระเบียบต้องถูกบังคับใช้ด้วย Code (Infrastructure as Code) ไม่ใช่อาศัยความจำหรือการตรวจสอบด้วยมนุษย์เพียงอย่างเดียว

### รายละเอียดและแนวทางปฏิบัติ

มนุษย์มีความผิดพลาดได้ (Human Error) และมาตรฐานการรีวิวโค้ดอาจไม่สม่ำเสมอ เครื่องมือ Static Analysis จะต้องทำหน้าที่เป็น Gatekeeper ที่ไม่ยอมให้โค้ดคุณภาพต่ำเข้าสู่ Repository

1. **CI/CD Integration:** Pipeline ต้องล้มเหลว (Fail) ทันทีหากพบ Warning หรือ Error จาก Linter
2. **Zero Tolerance:** ไม่อนุญาตให้มี Warning หลงเหลือใน Codebase

### Implementation Guidelines

**Tools Requirement:**

* **Python:** `Ruff` (Strict mode), `MyPy` (Disallow Any)
* **TypeScript:** `ESLint` (Strict Rules), `Prettier`
* **Go:** `GolangCI-Lint` (Enable `errcheck`, `gocyclo`, `gosec`)

**Workflow:**

1. **Pre-commit Hook:** ตรวจสอบโค้ดในเครื่องนักพัฒนาก่อน Commit
2. **Pull Request Check:** ระบบ CI ตรวจสอบซ้ำ หากไม่ผ่าน ห้าม Merge โดยเด็ดขาด

---

**บทสรุป**
การปฏิบัติตามมาตรฐานทั้ง 10 ข้อนี้อย่างเคร่งครัด จะช่วยสร้างรากฐานทางวิศวกรรมที่แข็งแกร่ง ลดความเสี่ยงในการเกิดข้อผิดพลาดร้ายแรง และทำให้ทีมพัฒนาสามารถส่งมอบซอฟต์แวร์ที่มีคุณภาพสูงได้อย่างต่อเนื่อง แม้จะมีการเปลี่ยนแปลงบุคลากรก็ตาม
