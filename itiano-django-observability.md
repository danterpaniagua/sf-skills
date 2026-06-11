---
name: django-observability
description: Analyze Django/PostgreSQL code and add operational logging focused on observability, troubleshooting, and production support.
---

# Mission

Improve operational visibility while avoiding unnecessary log noise.

Every proposed log must have a clear operational purpose.

Logs should help developers, operators, and SREs answer:

* What happened?
* When did it happen?
* Who was affected?
* Which operation failed?
* How can it be investigated?

---

# Analysis First

Before adding logs:

1. Analyze the business flow.
2. Analyze the request lifecycle.
3. Identify critical operations.
4. Identify failure points.
5. Identify PostgreSQL interactions.
6. Review existing logging.
7. Detect observability gaps.

Produce:

# Logging Analysis

## Business Flow

## Critical Operations

## Failure Points

## Existing Logging

## Observability Gaps

Do not modify code before completing this analysis.

---

# Logging Objectives

Logs should support:

* Incident investigation
* Production troubleshooting
* Performance analysis
* Audit of important business events
* Root cause analysis

Logs should not be added solely for debugging.

---

# Never Log

Never log:

* Passwords
* Tokens
* JWTs
* API Keys
* Secrets
* Authorization headers
* Cookies
* Session identifiers
* Credit card information
* Personal data unless explicitly justified

Mask sensitive values whenever possible.

---

# Logging Levels

## INFO

Business-relevant events.

Examples:

* User created
* User authenticated
* Order created
* Payment completed
* Report generated

---

## WARNING

Unexpected but recoverable situations.

Examples:

* Retry attempts
* Missing optional data
* Slow operations
* Temporary service degradation

---

## ERROR

Failures requiring investigation.

Examples:

* Unhandled exceptions
* Database failures
* External service failures
* Transaction rollbacks

---

# Django Logging Rules

Use project loggers.

Prefer:

```python
logger.info(
    "order_created",
    extra={
        "order_id": order.id,
        "user_id": request.user.id,
    }
)
```

Avoid:

```python
print("order created")
```

Avoid:

```python
logger.info(f"Order {order.id} created")
```

Prefer structured context.

---

# Correlation Rules

Whenever available include:

* request_id
* correlation_id
* user_id
* tenant_id
* endpoint
* operation

Logs should allow reconstruction of a complete request flow.

---

# ORM Query Observability

When investigating slow or unexpected database behaviour from Django:

* Use `str(queryset.query)` to inspect the generated SQL without hitting the database.
* Enable `django.db.connection.queries` (requires `DEBUG=True`) to capture all queries for a request — check `len(connection.queries)` and look for duplicates.
* N+1 pattern: repeated identical queries differing only by PK → missing `select_related` or `prefetch_related`.
* Log query count and total duration at the end of critical views or tasks.

---

# PostgreSQL Logging Rules

Log:

* Slow operations
* Bulk updates
* Bulk inserts
* Transaction failures
* Deadlocks
* Lock contention
* Timeout events

Include:

* operation
* duration_ms
* affected_rows
* model_name

Do not log:

* Full query results
* Sensitive values
* Raw SQL containing personal information

---

# Performance Protection

Avoid logging inside loops.

Avoid:

```python
for item in items:
    logger.info(...)
```

Prefer:

```python
logger.info(
    "items_processed",
    extra={
        "count": len(items)
    }
)
```

Use summary logging whenever possible.

---

# Exception Logging

Prefer:

```python
logger.exception(
    "invoice_generation_failed",
    extra={
        "invoice_id": invoice.id,
        "user_id": user.id,
    }
)
```

Avoid:

```python
logger.error(str(error))
```

Exception logs should preserve operational context.

---

# Incident Readiness Review

For every proposed logging change evaluate:

If this fails at 03:00 AM:

Can we determine:

* What happened?
* Which request failed?
* Which user was involved?
* Which endpoint failed?
* Which operation failed?
* Which exception occurred?
* Whether PostgreSQL was involved?
* How long the operation took?

If any answer is "No", propose additional logging.

---

# Output

# Logging Plan

## Missing Observability

## Logs To Add

### INFO

### WARNING

### ERROR

## PostgreSQL Observability

## Security Review

## Expected Operational Value

## Noise Assessment

## Incident Readiness Assessment

---

# Scope Protection

Do not introduce:

* Refactors
* Business logic changes
* Architecture changes

unless explicitly requested.

Focus exclusively on observability improvements.

---

# Completion Criteria

A logging task is complete only when:

* Observability gaps were identified.
* Logging proposals are justified.
* Sensitive information remains protected.
* Log noise remains acceptable.
* Incident readiness was evaluated.
* Repository standards remain satisfied.
