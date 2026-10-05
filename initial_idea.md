Proposed architecture
                    ┌─────────────────┐
                    │   data/         │
                    │                 │
                    │ rules.pdf       │
                    │       OR        │
                    │ rules.txt       │
                    └────────┬────────┘
                             │
                             ▼
                 ┌──────────────────────┐
                 │ Rule Extractor       │
                 │                      │
                 │ PDF/TXT              │
                 │       ↓              │
                 │ Structured rules     │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │ rules.json           │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │ Schedule Engine      │
                 │                      │
                 │ Current IST          │
                 │ ETD                  │
                 │ Rules                │
                 │ Delivery Status      │
                 └──────────┬───────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │ Rule Match? │
                     └──────┬──────┘
                            │
                    ┌───────┴───────┐
                    │               │
                   YES              NO
                    │               │
                    ▼               ▼
                 TRUE            FALSE

The JSON should be more structured
I would not keep the JSON exactly as your sample. We should distinguish between:
1. ETD information
2. Pre-ETD reminder rules
3. Post-ETD rules
4. Post-ETD stopping/expiry conditions
5. Delivery status

For example:
{
  "etd": {
    "date": "2026-11-15",
    "timezone": "Asia/Kolkata"
  },

  "pre_etd_rules": [
    {
      "days_before_etd": 30
    },
    {
      "days_before_etd": 15
    },
    {
      "days_before_etd": 3
    }
  ],

  "post_etd_rules": {
    "enabled": true,

    "reminder_schedule": [
      {
        "days_after_etd": 1
      },
      {
        "days_after_etd": 3
      },
      {
        "days_after_etd": 6
      }
    ],

    "stop_condition": {
      "when_delivered": true,
      "maximum_days_after_etd": 30
    }
  }
}

This is flexible enough to represent:
ETD - 30 days
ETD - 15 days
ETD - 3 days

ETD + 1 day
ETD + 3 days
ETD + 6 days

The critical concept: Delivery Status Gate
This is the part I would make explicit in the architecture.
Once ETD passes, the system enters:
POST_ETD_PENDING

It does not immediately send anything.
Instead:
ETD passed
     ↓
Ask user:
"Has the product been delivered?"
     ↓
 ┌───────────────┐
 │               │
YES             NO
 │               │
 ▼               ▼
STOP        Evaluate post-ETD rules
                │
                ▼
        Is today a reminder day?
                │
          ┌─────┴─────┐
          │           │
         YES          NO
          │           │
          ▼           ▼
       TRUE          FALSE

For example.
So if the rules say:
"reminder_schedule": [
    {"days_after_etd": 1},
    {"days_after_etd": 3},
    {"days_after_etd": 6}
]

and the product isn't delivered:
Day	Delivery status	Rule	Trigger
ETD + 1	No	Matches	true
ETD + 2	No	No match	false
ETD + 3	No	Matches	true
ETD + 4	No	No match	false
ETD + 5	No	No match	false
ETD + 6	No	Matches	true


If the user says YES on ETD + 4:
Delivery = YES
       ↓
Post-ETD process terminates
       ↓
No more triggers

3. I recommend separating "rule evaluation" from "user interaction"
This will make the implementation much cleaner.
Rule Engine
Its only job is:
Given ETD + current date + rules + delivery status, should an email be triggered?

Conceptually:
should_send_email(    etd,    current_datetime,    rules,    delivery_status) -> bool


It returns only:
True

or
False

cheduler
The scheduler is responsible for waking the system up at the appropriate time.
For example:
Every day
    ↓
Get current IST date
    ↓
Evaluate rules
    ↓
Return Boolean

The scheduler shouldn't contain the business rules itself.

One important improvement: don't make the JSON the permanent source of truth
I'd use:
PDF/TXT
   ↓
Extraction
   ↓
Normalized rules.json
   ↓
Validation
   ↓
Schedule Engine

The PDF remains the human source of truth.
The JSON is the compiled/normalized representation.
That gives us a very nice future workflow:
rules.pdf changed
       ↓
re-extract
       ↓
validate
       ↓
replace rules.json
       ↓
scheduler automatically uses new rules

No scheduler code needs to change when the company's reminder policy changes.

                    rules.pdf
                       │
                       ▼
                rule_extractor
                       │
                       ▼
                  rules.json
                       │
                       ▼
                rule_validator
                       │
                       ▼
                schedule_engine
                       │
              ┌────────┴────────┐
              ▼                 ▼
        pre-ETD logic      post-ETD logic
                                │
                                ▼
                       delivery_status
                                │
                                ▼
                         Boolean trigger

Yes. I agree with your direction, with one important refinement: **Docling should handle document understanding/extraction, while the LLM should only perform semantic normalization into your Pydantic schema. Everything after `rules.json` should be deterministic.**

I would keep this deliberately small. No database, no LangChain, no agent framework, no LLM in the evaluation loop.

## Recommended stack

| Layer | Technology | Purpose |
|---|---|---|
| Language | **Python 3.14** | Entire application |
| Document extraction | **Docling** | PDF/TXT → clean structured text |
| Rule interpretation | **Grok / `gpt-oss-8b`** | Text → structured JSON |
| Data model | **Pydantic v2** | Define the JSON schema |
| JSON | **JSON** | Persist extracted rules |
| Date/time | **Python `datetime` + `zoneinfo`** | Deterministic IST handling |
| Scheduling | **APScheduler** | Wake the evaluator periodically |
| Evaluation | **Pure Python** | `True` / `False` only |
| File safety | **Atomic file replacement** | Prevent corrupt `rules.json` |
| Logging | **Python `logging`** | Observability |
| Testing | **pytest** | Deterministic rule-engine tests |
| Config | **`.env` + `pydantic-settings`** | API/configuration |
| Packaging | **uv** | Python 3.14 environment/dependencies |

### Architecture

```text
                    rules.pdf / rules.txt
                            │
                            ▼
                       ┌─────────┐
                       │ Docling │
                       └────┬────┘
                            │
                       extracted text
                            │
                            ▼
                    ┌────────────────┐
                    │  Grok /        │
                    │  gpt-oss-8b    │
                    │                │
                    │ ONLY HERE      │
                    └───────┬────────┘
                            │
                     structured JSON
                            │
                            ▼
                       Pydantic
                       Rule model
                            │
                            ▼
                  atomic rules.json write
                            │
                            ▼
              ┌───────────────────────────┐
              │      SCHEDULER            │
              │                           │
              │ current IST time          │
              │            ↓              │
              │ evaluate_schedule(...)    │
              │            ↓              │
              │       True / False        │
              └───────────────────────────┘
```

The LLM effectively becomes a **one-time compiler** from human policy language into machine rules.

---

# The interesting part: your rules need intervals

Your latest example exposes something important.

A rule might say:

> ETD: 10:00 AM. Send an email every hour before ETD and every minute after ETD for one hour.

Or:

> Send reminders every minute after ETD.

Or:

> Send reminders every 3 days after ETD.

So I would **not** model rules as merely:

```python
days_before_etd: int
```

We need a generic temporal rule representation.

I'd use something along these lines:

```python
class ScheduleRule(BaseModel):
    id: str

    phase: Literal[
        "before_etd",
        "after_etd"
    ]

    start_offset: timedelta

    end_offset: timedelta | None = None

    interval: timedelta | None = None

    action: Literal["send_email"]

    until: Literal["delivered"] | None = None
```

Conceptually:

```text
phase
    │
    ├── before_etd
    └── after_etd

start_offset
    │
    └── when does the rule begin?

end_offset
    │
    └── when does it stop?
       None = no explicit end

interval
    │
    └── how frequently does it trigger?

action
    │
    └── send_email

until
    │
    └── delivered
```

---

# Example 1

Suppose the file says:

> Send an email every hour starting 5 hours before ETD.

The JSON could become:

```json
{
  "etd": "2026-11-15T10:00:00+05:30",
  "rules": [
    {
      "id": "pre_etd_hourly",
      "phase": "before_etd",
      "start_offset": "-PT5H",
      "end_offset": null,
      "interval": "PT1H",
      "action": "send_email",
      "until": null
    }
  ]
}
```

---

# Example 2

Your second example:

> Send an email every minute after ETD for one hour.

```json
{
  "etd": "2026-11-15T10:00:00+05:30",
  "rules": [
    {
      "id": "post_etd_every_minute",
      "phase": "after_etd",
      "start_offset": "PT0M",
      "end_offset": "PT1H",
      "interval": "PT1M",
      "action": "send_email",
      "until": "delivered"
    }
  ]
}
```

This represents:

```text
10:00  ← ETD
10:01  ← trigger
10:02  ← trigger
10:03  ← trigger
...
10:59  ← trigger
11:00  ← trigger
11:01  ← STOP
```

If the user says:

```text
delivered = true
```

the rule immediately stops.

---

# Example 3: irregular days

Your earlier example:

> ETD + 1 day, +3 days, +6 days.

This actually suggests that we should support **two rule types**.

### Interval rule

```text
every 1 minute
every 1 hour
every 3 days
```

### Offset rule

```text
1 day after ETD
3 days after ETD
6 days after ETD
```

So I'd make the model slightly more explicit:

```python
class ScheduleRule(BaseModel):
    id: str

    phase: Literal["before_etd", "after_etd"]

    offsets: list[timedelta] | None = None

    start_offset: timedelta | None = None
    end_offset: timedelta | None = None
    interval: timedelta | None = None

    action: Literal["send_email"]

    until: Literal["delivered"] | None = None
```

Then:

### Specific dates

```json
{
  "offsets": [
    "P1D",
    "P3D",
    "P6D"
  ]
}
```

### Repeating schedule

```json
{
  "start_offset": "PT0M",
  "end_offset": "PT1H",
  "interval": "PT1M"
}
```

This gives the LLM enough vocabulary to translate quite a wide range of policy language.

---

# But I would make one architectural change

Don't make `APScheduler` responsible for determining whether a rule fires.

Its job should only be:

> **Wake the application up.**

For example:

```text
APScheduler
     │
     │ every 1 minute
     ▼
evaluation loop
     │
     ▼
evaluate_schedule()
     │
     ▼
True / False
```

The actual decision belongs entirely to your deterministic function.

```python
trigger = evaluate_schedule(
    current_time,
    etd,
    rules,
    delivery_status
)
```

And:

```python
assert trigger in (True, False)
```

That's your core.

---

# How I'd handle the one-minute case

I'd probably make the scheduler run at the **smallest practical resolution you need**, which in your current design is one minute.

```text
Every minute
     ↓
read current IST
     ↓
load rules.json
     ↓
load delivery status
     ↓
evaluate_schedule()
     ↓
True / False
```

You don't want the scheduler dynamically creating hundreds of scheduled jobs based on whatever the LLM generated.

That's unnecessary complexity.

Instead:

**One evaluation loop + deterministic rule evaluation.**

This is much easier to reason about.

---

# Time handling

Definitely use:

```python
from datetime import datetime
from zoneinfo import ZoneInfo

IST = ZoneInfo("Asia/Kolkata")

now = datetime.now(IST)
```

Do **not** manually add/subtract `5:30`.

And I'd store ETD as an ISO-8601 timestamp including the timezone:

```text
2026-11-15T10:00:00+05:30
```

That makes the temporal model explicit.

---

# Pydantic's role

I agree with your proposed use.

Pydantic isn't going to "validate the business rules."

It's simply going to give us a **strong internal representation**.

```text
LLM
 │
 ▼
JSON
 │
 ▼
Pydantic model
 │
 ▼
rules.json
```

So:

```python
rules = Rules.model_validate(llm_output)
```

isn't saying:

> "Is this company's policy logically correct?"

It's saying:

> "Can my program represent what the LLM gave me in the structure I expect?"

That's an important distinction.

---

# Atomic JSON writing

Absolutely.

Don't do:

```python
open("rules.json", "w")
```

and start writing directly to the production file.

Instead:

```text
generated rules
      ↓
rules.tmp
      ↓
flush
      ↓
fsync
      ↓
atomic replace
      ↓
rules.json
```

On successful completion:

```text
rules.json ← complete new version
```

If the process crashes halfway through:

```text
rules.json ← old valid version
```

That is exactly what we want.

---

# Project structure

I'd make the first version:

```text
schedule_followup/
│
├── data/
│   └── rules.txt
│
├── config/
│   └── rules.json
│
├── src/
│   └── schedule_followup/
│       │
│       ├── __init__.py
│       │
│       ├── models.py
│       │
│       ├── extractor.py
│       │
│       ├── rule_parser.py
│       │
│       ├── scheduler.py
│       │
│       ├── evaluator.py
│       │
│       ├── delivery_status.py
│       │
│       ├── storage.py
│       │
│       └── main.py
│
├── tests/
│   ├── test_evaluator.py
│   ├── test_intervals.py
│   ├── test_offsets.py
│   └── test_delivery_status.py
│
├── .env
├── pyproject.toml
└── README.md
```

### Responsibilities

**`extractor.py`**

```text
PDF/TXT
 ↓
Docling
 ↓
clean text
```

**`rule_parser.py`**

```text
clean text
 ↓
Grok / gpt-oss-8b
 ↓
structured rule JSON
```

This is the **only LLM-containing module**.

**`models.py`**

Pydantic models.

**`storage.py`**

Atomic JSON reads/writes.

**`scheduler.py`**

Periodic wake-up.

**`evaluator.py`**

The sacred little deterministic engine:

```python
evaluate_schedule(...) -> bool
```

**`delivery_status.py`**

For now:

```python
delivered: bool
```

Later this can be connected to your UI/workflow.

---

# Dependencies

I'd keep `pyproject.toml` lean:

```toml
[project]
requires-python = ">=3.14"

dependencies = [
    "docling",
    "pydantic",
    "pydantic-settings",
    "apscheduler",
    "python-dotenv",
]
```

Then whatever package/API client you're using for the Grok / `gpt-oss-8b` endpoint.

For development:

```toml
[dependency-groups]
dev = [
    "pytest",
    "pytest-cov",
    "ruff",
]
```

And I'd use **uv** rather than pip/conda for this project.

```text
uv
 ↓
Python 3.14
 ↓
isolated project environment
 ↓
locked dependencies
```

---

# One thing I would NOT add

I would explicitly avoid:

```text
LangChain       ❌
LangGraph       ❌
PandasAI        ❌
Celery          ❌
Redis           ❌
PostgreSQL      ❌
Vector DB       ❌
Agent framework ❌
LLM evaluator   ❌
```

At this stage they're architectural ballast.

Your problem is fundamentally:

```text
Document → Structured temporal rules → Deterministic temporal evaluation
```

It doesn't need an agent swarm wearing tiny business suits. 😄

---

## Final architecture

I'd lock the architecture at:

```text
                    ┌────────────────┐
                    │ rules.pdf/txt  │
                    └───────┬────────┘
                            │
                         Docling
                            │
                            ▼
                     Extracted Text
                            │
                            ▼
                    ┌────────────────┐
                    │ Grok           │
                    │ gpt-oss-8b     │
                    │                │
                    │ ONLY LLM USE   │
                    └───────┬────────┘
                            │
                            ▼
                     Structured JSON
                            │
                         Pydantic
                            │
                            ▼
                    Atomic rules.json
                            │
                            │
            ┌───────────────┴───────────────┐
            │                               │
            ▼                               ▼
      Current IST time                Delivery status
            │                               │
            └───────────────┬───────────────┘
                            ▼
                 ┌────────────────────┐
                 │ evaluate_schedule() │
                 │                    │
                 │ DETERMINISTIC      │
                 │                    │
                 │       bool         │
                 └─────────┬──────────┘
                           │
                    ┌──────┴──────┐
                    ▼             ▼
                  TRUE          FALSE
                    │             │
                    ▼             ▼
             Email trigger      Nothing
```

**Tech stack:** Python 3.14 + Docling + Grok/`gpt-oss-8b` + Pydantic + `datetime/zoneinfo` + APScheduler + atomic JSON storage + pytest + uv.

And the most important boundary is:

> **LLM ends at `rules.json`. From that point onward, the system is deterministic code.**

That is the architecture I'd use for the first implementation.