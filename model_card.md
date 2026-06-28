# BugHound Mini Model Card (Reflection)

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts.

---

## 2) How does it work?

BugHound runs a five-step workflow:

1. **PLAN** — Logs intent; no real decision-making here, just sets up the run.
2. **ANALYZE** — Detects issues. In heuristic mode it pattern-matches for `print(`, bare `except:`, and `TODO`. In Gemini mode it sends the code to the API with a structured JSON prompt and parses the response; if the response is empty or unparseable it falls back to heuristics.
3. **ACT** — Proposes a fix. Heuristic mode applies regex transforms keyed on issue `type` labels (`"Reliability"` → fix bare except, `"Code Quality"` → swap print for logging). Gemini mode passes the issue list and original code to a fixer prompt and returns the raw code output.
4. **TEST** — Calls `assess_risk()` in `reliability/risk_assessor.py`. Scores the fix from 0–100 by deducting points for issue severity, multiple concurrent issues, structural changes (shorter code, missing returns, modified bare excepts), and comment-only output.
5. **REFLECT** — Decides whether to recommend auto-apply (`should_autofix`) based on the risk level. Only `"low"` (score ≥ 75) triggers auto-fix.

Heuristics handle detection and fixing fully offline. Gemini provides richer, more context-aware analysis when an API key is available, but the agent always falls back to heuristics if the model is unavailable or returns unusable output.

---

## 3) Inputs and outputs

**Inputs tested:**

- `cleanish.py` — short, already-correct function using `logging`; no issues expected
- `flaky_try_except.py` — a file-reading function with a bare `except:` and an unclosed file handle
- `mixed_issues.py` — a function with a `print()`, a bare `except:`, and a `TODO` comment; three distinct issue types
- Weird case — a single-line comment file (`# This file is intentionally left blank`)
- Minimal synthetic case — `x = 1\nprint(x)` — short, no return, one low-severity issue

**Outputs observed:**

| Input | Issues detected | Fix proposed | Risk level |
|---|---|---|---|
| cleanish.py | None | Original returned unchanged | low / score 100 |
| flaky_try_except.py (heuristic) | 1 — bare except (High) | Replaced `except:` with `except Exception as e:` | high (score 5 — heuristic fixer also removed return) |
| flaky_try_except.py (Gemini direct) | 3 — bare except, unclosed file handle, silent error swallowing | LLM rewrote with `with open(...)` and specific exception | medium (score 55) |
| mixed_issues.py | 3 — print, bare except, TODO | Heuristic: added `import logging`, swapped print → logging.info, rewrote except | high (score 0) |
| Weird comment-only | None | Original returned unchanged | low / score 100 |
| Short print case | 1 — print (Low) | MockClient returned a comment string | **should_autofix: True before guardrail; False after** |

---

## 4) Reliability and safety rules

### Rule 1: Return statement removal (`–30 points`)
**What it checks:** Whether `return` appeared in the original but is absent in the fix.  
**Why it matters:** Removing a return statement changes program behavior — callers that depend on the return value will get `None` instead, which can cause subtle bugs far from the change site.  
**False positive:** A fix that legitimately refactors a function to use a helper and moves the return inside the helper would trigger this even though no logic was lost.  
**False negative:** If the fix replaces `return x` with `return None` explicitly, the check passes even though the meaningful value was removed.

### Rule 2: Comment-only fix (`–40 points`) *(added this session)*
**What it checks:** Whether the fixed code contains any executable (non-blank, non-comment) lines.  
**Why it matters:** A model or mock client can return a comment or refusal string that is non-empty and passes the length check — the agent would accept it as a valid fix. A comment-only output cannot be correct Python code for any real function.  
**False positive:** A fix that is intentionally a module-level docstring or a file that is legitimately only constants and comments could be flagged (edge case, but possible).  
**False negative:** A fix that contains one real line of code (e.g., `pass`) plus a comment would pass the check, even if `pass` is semantically wrong.

### Rule 3: Multiple-issue penalty (`–15 points`) *(added this session)*
**What it checks:** Whether two or more issues were found.  
**Why it matters:** A fix addressing three separate problems touches more code paths and is harder to verify manually — the chance of an unintended interaction is higher.  
**False positive:** A file with two trivially independent issues (e.g., two unrelated print statements) is penalized even though the fix is still straightforward.  
**False negative:** A single high-severity issue that requires a large structural rewrite is not penalized by this rule, even though the change is more complex.

---

## 5) Observed failure modes

**Failure 1 — BugHound missed an issue it should have caught**

File: `flaky_try_except.py`

```python
def load_text_file(path):
    try:
        f = open(path, "r")
        data = f.read()
        f.close()
    except:
        return None
    return data
```

In heuristic mode, BugHound only flagged the bare `except:`. It missed that `f.close()` inside a try block is never reached if `f.read()` raises — the file handle leaks. Gemini caught this and suggested replacing the whole block with `with open(path, "r") as f:`. The heuristic has no pattern for resource management, so this class of issue is invisible to offline mode.

**Failure 2 — BugHound suggested a fix that felt risky or unnecessary**

File: short synthetic case — `x = 1\nprint(x)\n`

The MockClient returned `# MockClient: no rewrite available in offline mode.` as the proposed fix. Before the comment-only guardrail was added, the risk assessor scored this 95/low and set `should_autofix: True`. The system was ready to replace two lines of valid Python with a comment — effectively deleting the file's content. The length check and return-statement check both failed to catch it because the original file was also very short and had no return statement.

---

## 6) Heuristic vs Gemini comparison

**What Gemini detected that heuristics did not:**
- Unclosed file handle / resource leak (missing `with` statement) in `flaky_try_except.py`
- Silent error swallowing: returning `None` without logging hides the original exception type from callers
- Specific exception suggestions (`FileNotFoundError`, `IOError`) rather than just "use `except Exception`"

**What heuristics caught consistently:**
- Bare `except:` blocks (reliable regex pattern)
- `print()` statements
- `TODO` comments
These three patterns are fast, offline, and produce stable, labeled results every time.

**How proposed fixes differed:**
- Heuristic fixer applies mechanical transforms: it always adds `import logging`, always replaces `print(` globally, always uses the same `except Exception as e:` boilerplate. It does not read the surrounding logic.
- Gemini rewrote code contextually: it used `with open(...)` instead of manual close, caught specific exceptions, and produced code that actually made semantic sense for the function.

**Did the risk scorer agree with intuition?**
Mostly yes — high-severity issues correctly blocked auto-fix. But the scorer was fooled by the MockClient comment-only output on a short file (score 95), which did not match intuition at all. The multiple-issue penalty and comment-only guardrail both added caution in cases where the raw score felt overconfident.

---

## 7) Human-in-the-loop decision

**Scenario:** The user runs BugHound on a file with a bare `except:` block (High severity). The agent proposes replacing it with `except Exception as e:`. The risk assessor scores it as medium (score 55) and sets `should_autofix: False`.

However, even if the score were higher, a high-severity issue means the original code had a significant reliability problem — any automated rewrite of exception-handling logic deserves a human eye, because the correct exception type depends on the calling context.

**Trigger:** If any issue in the list has `severity == "High"`, block auto-fix unconditionally, regardless of score.

**Where to implement:** `risk_assessor.py` — it is the dedicated safety layer and the right place for policy rules. The check would sit just before the `should_autofix` assignment:
```python
has_high_severity = any(str(i.get("severity","")).lower() == "high" for i in issues)
should_autofix = level == "low" and not has_high_severity
```

**Message to show the user:**
> "A high-severity issue was found. Auto-fix is disabled — please review the proposed change before applying it."

---

## 8) Improvement idea

**More robust output parsing: require the model to return a structured envelope**

Currently the analyzer prompt asks for a bare JSON array. Models sometimes wrap output in code fences, add explanations, or use different field names (`"Error Handling"` instead of `"Reliability"`). Each variation silently triggers a fallback to heuristics.

A small improvement: change the prompt to ask for a single JSON object `{"issues": [...]}` instead of a bare array. This makes the response easier to locate by key lookup rather than bracket-scanning, and it is less likely to be confused with inline JSON in the code being analyzed. Add a normalization step that maps known variant labels (`"Error Handling"` → `"Reliability"`, `"Resource Management"` → `"Reliability"`) to the canonical set the heuristic fixer understands. This closes the gap where Gemini detects an issue correctly but the heuristic fixer silently skips the fix because the type label does not match exactly — without adding a new model call or significant complexity.
