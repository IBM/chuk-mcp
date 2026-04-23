# chuk-mcp Technical Roadmap

Low-priority technical debt items identified during the April 2026 codebase review.
High and medium priority items have already been addressed.

---

## 1. Resolve Dual `JSONRPCMessage` Definition

**File:** `src/chuk_mcp/protocol/messages/json_rpc_message.py`
**Effort:** 2–4 hours

`JSONRPCMessage` is defined twice in the same module:

- Line 66: as a `Union` type alias over `JSONRPCRequest | JSONRPCNotification | JSONRPCResponse | JSONRPCError`
- Line 259: as a concrete `McpPydanticBase` class

Python silently resolves all references to the class (line 259), making the Union alias unreachable.
This causes 15+ `# type: ignore` comments in `__init__.py` and related files, and confuses mypy.

**Recommended fix:**

1. Rename the Union alias to `AnyJSONRPCMessage` (or similar)
2. Update all call sites that currently call `.model_validate()` on the Union type
3. Remove the `# type: ignore` comments that are no longer necessary

---

## 2. Reduce `mcp_pydantic_base.py` Complexity

**File:** `src/chuk_mcp/protocol/mcp_pydantic_base.py`
**Effort:** 4–6 hours

The Pydantic fallback implementation is 883 lines — almost as complex as Pydantic itself — and carries several risks:

- `get_type_hints()` is called on every `model_validate()` invocation (no caching)
- Type resolution uses 4 different fallback strategies (lines 142–187), making it hard to reason about
- A hard-coded `str | int` union special-case (line 271) means new union types require code changes
- `_deep_validate()` has no recursion depth limit (line 322)
- Broad `except Exception: pass` swallows errors silently (lines 426, 718)

**Recommended approaches (pick one):**

A. **Cache `get_type_hints()` per class** — lowest effort, biggest performance gain
B. **Extract type resolution into a dedicated module** with clear strategy pattern
C. **Vendor the official MCP SDK's fallback** if one exists upstream
D. **Make Pydantic a required dependency** if the fallback adds more risk than it saves

---

## 3. Add Concurrency and Load Tests

**Files:** `tests/`
**Effort:** 2–3 hours

The current test suite (1,279 tests) covers the happy path well but has no:

- Concurrent message routing tests (multiple in-flight requests)
- Large payload tests (messages near or above buffer size 100)
- Long-running connection stability tests
- Load / throughput benchmarks

These would be valuable before deploying in high-throughput environments.

---

## 4. Structured Logging

**Files:** various
**Effort:** 2 hours

Logging is inconsistent across modules — varying levels, formats, and context. Consider:

- Using `structlog` or a consistent `extra={}` pattern for machine-parseable logs
- Standardising on `%s` formatting (avoid f-strings in log calls to defer interpolation)
- Adding a request-id or correlation-id field to message-processing log lines
