# Documentation Guidelines

> Docs are code. Stale docs are bugs.

---

## 1. What Must Be Documented

| Surface            | When required                                      |
|--------------------|----------------------------------------------------|
| Public function    | Always — docstring on every exported symbol        |
| Internal helper    | When non-obvious logic warrants explanation        |
| User-facing change | README or `CHANGELOG.md` updated in the same PR   |
| New service / tool | `docs/index.md` + `docs/quickstart.md` before GA  |
| Architecture decision | ADR in `docs/adr/` (see architecture guidelines) |

---

## 2. Docstring Format by Language

Every docstring answers **three questions in order**:
1. What does it do? *(one line)*
2. What are the inputs, outputs, and errors?
3. How do I call it? *(at least one example for non-trivial APIs)*

```python
# Python — Google style
def fetch_order(order_id: str) -> Order:
    """Retrieve a single order by its ID.

    Args:
        order_id: UUID of the order to fetch.

    Returns:
        The matching Order domain object.

    Raises:
        OrderNotFoundError: If no order exists with that ID.

    Example:
        order = fetch_order("ord_abc123")
    """
```

```typescript
// TypeScript — TSDoc
/**
 * Retrieve a single order by its ID.
 * @param orderId - UUID of the order to fetch.
 * @returns The matching Order domain object.
 * @throws {OrderNotFoundError} If no order exists with that ID.
 * @example
 *   const order = await fetchOrder('ord_abc123');
 */
```

---

## 3. Markdown Docs Site Structure

```
  docs/
  ├── index.md        ← what, who, where to get help
  ├── quickstart.md   ← clone → install → run in < 10 min
  ├── architecture.md ← C4-Context + C4-Container + failure modes
  ├── runbook.md      ← paging, alerts, rollback
  ├── api.md          ← generated from OpenAPI/proto, not hand-written
  └── adr/
      └── 0001-*.md
```

Every page follows this structure:
1. **H1 title** (one per page, matches slug)
2. **One-paragraph intro** — what is this and why does it exist?
3. **Body** — `##` H2 sections, `###` H3 only when truly necessary
4. **See also** — cross-links at the bottom

---

## 4. Diagrams

- Use **Mermaid** for inline diagrams (renders on GitHub and Astro Starlight).
- Commit Excalidraw sources (`.excalidraw` + exported PNG) for richer visuals.
- **No screenshots of code.** Code belongs in fenced code blocks with a language tag.

---

## 5. Code Examples

- Always specify the language: ` ```python `, never bare ` ``` `.
- Examples must be **copy-pasteable and runnable as-is**. If setup is needed, link to the quickstart.
- Max 30 lines per block. Longer examples → `docs/examples/<name>.<ext>`.

---

## 6. User-Facing Changes

Any change visible to end users (new feature, behaviour change, deprecation, breaking change) must update **both**:
- `README.md` (or the relevant `docs/` page)
- `CHANGELOG.md` under the `[Unreleased]` section using [Keep a Changelog](https://keepachangelog.com) format

A PR that ships user-facing changes without a doc update **will not be merged**.

---

## 7. Anti-Patterns

- ❌ `// TODO: add docs` committed to `main`
- ❌ Docstrings that restate the signature in prose
- ❌ Stale examples that no longer compile or run
- ❌ Architecture docs that describe the system as it was, not as it is

---

## PR Gate Checklist

- [ ] All new public functions have docstrings
- [ ] User-facing changes update README / `CHANGELOG.md`
- [ ] New service ships `docs/index.md` + `docs/quickstart.md`
- [ ] No stale examples introduced
- [ ] Diagrams use Mermaid or committed Excalidraw sources
