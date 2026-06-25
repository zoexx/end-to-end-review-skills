# Contributing

Thanks for helping make these review skills sharper. Two kinds of contributions are especially welcome:

1. **New reference guides** inside an existing skill's `reference/` (e.g. a new framework or language guide under `frontend-review/reference/`).
2. **New review layers** as whole new skills (e.g. `mobile-review`, `infra-review`, `performance-review`, `data-ml-review`).

## House style

So the skills compose and feel like one suite, every skill must:

- **Use the shared review model** — the [four-phase process](README.md#the-shared-review-model), the six [severity labels](README.md#severity-labels), and the four verdict states. Restate them in the `SKILL.md` (skills are installed independently, so each must be self-contained).
- **Follow the `SKILL.md` frontmatter convention:**

  ```yaml
  ---
  name: <skill-name>            # matches the folder name, kebab-case
  description: |                # third person; include a "Use when:" line of trigger phrases
    ...
    Use when: ...
  allowed-tools:                # read-only by default — reviews observe, they don't mutate
    - Read
    - Grep
    - Glob
    - Bash
    - WebFetch
  ---
  ```

- **Be specific, not generic.** A finding the reader couldn't have written themselves is the whole point. Prefer "wrap this callback in `useCallback` — it's recreated every render and breaks the memoized child's `props` equality" over "consider memoization."
- **Show the fix.** Reference guides pair a ❌ anti-pattern with a ✅ corrected version wherever it helps.
- **Keep `SKILL.md` lean; push depth into `reference/`.** The `SKILL.md` is the always-loaded entry point — it should be a navigable map. Long language/framework deep-dives belong in `reference/` files loaded on demand.

## Adding a new skill

1. Create `<skill-name>/` with a `SKILL.md`, a `reference/` directory, and an `assets/` checklist.
2. Register it in the root [README.md](README.md) table and, if it should participate in full sweeps, in `end-to-end-review/SKILL.md`'s routing table.
3. Open a PR describing what failure modes the new layer catches that the existing skills miss.

## Reporting gaps

If a skill missed something it should have caught, open an issue with the smallest diff that reproduces the miss. Real-world misses are the best fuel for new reference material.
