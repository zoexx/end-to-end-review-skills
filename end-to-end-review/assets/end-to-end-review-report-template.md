# End-to-end review — <PR title / branch name>

> Copy this template, fill it in, delete the guidance comments.

**Verdict:** <✅ approve | 💬 approve with comments | 🔁 request changes | ⛔ block>
**Layers engaged:** <frontend · backend · database · security>
**Scope reviewed:** `<base>...<head>` — <N files, +X/−Y lines>

**Summary:** <2–3 sentences. What does this change do, and what are the headline risks? Written so a busy author or PM gets the gist without scrolling.>

---

## 🔴 Blocking — must fix before merge

<!-- correctness, security, data-loss. If empty, write "None." -->
- 🔴 [<layer>] `path/to/file:line` — <one-line finding>.
  **Why:** <the concrete failure: the bug it causes, the attack it enables, the user it hurts.>
  **Fix:** <the specific change to make.>

## 🟠 Important — should fix

- 🟠 [<layer>] `path/to/file:line` — <finding>.
  **Why:** <impact.>
  **Fix:** <change.>

## 🟡 Nits — minor, non-blocking

- 🟡 [<layer>] `path/to/file:line` — <finding>.

## 🔵 Suggestions — optional improvements

- 🔵 [<layer>] `path/to/file:line` — <alternative worth considering>.

## 📚 Learning — context, no action required

- 📚 <a pattern or background worth knowing, linked to a line if relevant.>

## 🌟 Praise — good work worth calling out

- 🌟 [<layer>] <what was done well and why it's good.>

---

## Cross-layer findings

<!-- Issues that are only wrong because of how layers interact, owned by no single layer. -->
- <severity> **cross-layer** — <finding spanning ≥2 layers>.
  **Why:** <how the layers compound the problem.>
  **Fix:** <coordinated change, noting each layer's part.>

---

## By layer

| Layer | Verdict | 🔴 | 🟠 | 🟡 | 🔵 |
| --- | --- | --- | --- | --- | --- |
| frontend | <verdict> | 0 | 0 | 0 | 0 |
| backend  | <verdict> | 0 | 0 | 0 | 0 |
| database | <verdict> | 0 | 0 | 0 | 0 |
| security | <verdict> | 0 | 0 | 0 | 0 |

**Overall = worst layer verdict.** Any 🔴 in any layer blocks the merge.
