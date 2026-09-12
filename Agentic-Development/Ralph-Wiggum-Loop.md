## Ralph Wiggum Loop

**Ralph Wiggum** یک تکنیک برای **AI Coding / Agentic Development** است که در آن یک AI Agent به‌صورت مکرر و iterative اجرا می‌شود تا یک task را کامل کند.

ایده‌ی اصلی:

**Work → Check → Repeat → Done**

به‌جای اینکه یک Agent را با یک **long-running context** نگه داریم، Agent را چندین بار اجرا می‌کنیم. در هر iteration، Agent می‌تواند context جدیدی داشته باشد، اما **state پروژه روی disk باقی می‌ماند**.

### Basic Flow

```text
        Task / Specification
                ↓
           AI Agent
                ↓
          Modify Code
                ↓
        Run Tests / Check
                ↓
          Is it DONE?
          ↙         ↘
        No           Yes
        ↓             ↓
   New iteration     DONE
        ↓
    AI Agent
        ↓
      ...
```

### مهم‌ترین ایده

در Ralph Loop، **repository به‌عنوان external memory** عمل می‌کند.

یعنی به جای اینکه Agent مجبور باشد تمام conversation قبلی را به خاطر داشته باشد، اطلاعات مهم در چیزهایی مثل این‌ها باقی می‌ماند:

* Source Code
* Tests
* Git history / diff
* Specification
* Progress files
* TODOs
* Documentation

بنابراین:

```text
Agent Context
      ↓
    کوتاه‌تر
      ↓
Repository State
      ↓
   ماندگار
```

هر iteration می‌تواند با یک context تازه شروع شود و وضعیت فعلی پروژه را بخواند.

### مثال

فرض کنیم task این است:

> Build an authentication system.

Ralph Loop ممکن است این‌طور پیش برود:

```text
Iteration 1
→ Read specification
→ Create User model
→ Write tests

Iteration 2
→ Read current code
→ Implement password hashing
→ Run tests

Iteration 3
→ Implement login endpoint
→ Run tests

Iteration 4
→ Implement refresh token
→ Run tests

Iteration 5
→ Run full test suite
→ Find a bug
→ Fix it

Iteration 6
→ Verify requirements
→ DONE
```

### چرا به آن Ralph Wiggum می‌گویند؟

اسم آن از شخصیت **Ralph Wiggum** در *The Simpsons* گرفته شده است.

ایده‌ی طنز آن این است که Agent لازم نیست همیشه بسیار sophisticated باشد؛ مهم این است که **persistent باشد و مرتب تلاش کند، نتیجه را بررسی کند و دوباره ادامه دهد.**

### Completion Criteria

یکی از مهم‌ترین قسمت‌های Ralph Loop داشتن **objective completion criteria** است.

به‌جای اینکه فقط از Agent بپرسیم:

> "Are you done?"

باید معیارهای قابل‌بررسی داشته باشیم:

```text
✓ Tests pass
✓ Typecheck passes
✓ Linter passes
✓ Requirements are satisfied
✓ No known errors remain
```

این باعث می‌شود Agent بتواند تا حد زیادی به‌صورت **autonomous** کار کند.

### رابطه با Smart Zone / Dumb Zone

Ralph Loop را می‌توان در قالب زیر دید:

```text
       SMART ZONE
           ↓
       AI Agent
    reasoning / decisions
           ↓
       DUMB ZONE
           ↓
   Code / Tests / Git / Loop
    deterministic execution
           ↓
        Feedback
           ↓
       AI Agent
```

**Smart Zone:**
جایی که AI باید reasoning و decision-making انجام دهد.

**Dumb Zone:**
جایی که بهتر است کارها deterministic، ساده و قابل‌پیش‌بینی باشند؛ مثل اجرای tests، typecheck، lint و بررسی state پروژه.

### نکته مهم

Ralph Loop به این معنی نیست که:

> "Agent را رها کنیم تا هرچقدر خواست code تولید کند."

این روش زمانی بهتر کار می‌کند که:

1. Task به‌خوبی تعریف شده باشد.
2. Specification واضح باشد.
3. Acceptance criteria مشخص باشد.
4. Tests یا مکانیزم‌های verification وجود داشته باشند.
5. Agent بتواند state فعلی پروژه را بخواند.

**اصل کلیدی:**

> Don't rely on the Agent remembering everything.
> Make the repository and verification system carry the state and feedback.

در نتیجه، Ralph Wiggum Loop یک روش برای تبدیل **AI Coding** از یک interaction یک‌باره به یک **iterative, feedback-driven software development process** است.
