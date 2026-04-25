# Dependency Check

Scan the code for any references to external scripts, modules, or files not provided in
the current context. **Apply in both Standalone and Self-Audit modes.** In Self-Audit
mode this is especially important: the code you generated may reference helpers, config
files, or utilities you assumed exist — surface those assumptions before they become
invisible hallucinations in the output.

| Signal | Examples |
|---|---|
| Imports / requires | `import utils`, `from helpers import *`, `require('./config')` |
| Shell invocations | `subprocess.run(['other_script.py'])`, `os.system(...)`, `exec(...)` |
| File path references | `open('data/config.json')`, `source ./env.sh`, `include 'lib.php'` |
| Dynamic loading | `importlib.import_module(...)`, `require(variable)`, `eval(...)` |
| Entry-point calls | `if __name__ == '__main__': run()` calling functions defined elsewhere |

### If external dependencies are detected

**Stop before producing the audit report.** Output this notice:

```
⚠️ Dependency Notice

This script references the following external scripts or files that were not provided:

  • <dependency name> — <how it is used, e.g. "imported for auth helpers", "invoked as subprocess", "read as config">
  • <dependency name> — <how it is used>

Without these, the audit may miss issues related to:
  - Contracts between this script and its dependencies (incorrect assumptions about inputs/outputs)
  - Security vulnerabilities introduced via dependency behaviour
  - Bugs that only manifest in the combined execution context

How would you like to proceed?
  A) Continue the audit with only the code provided (I'll flag gaps where missing context limits the review)
  B) Share the additional scripts so I can audit them together for a complete picture
```

Wait for the user's response before proceeding.

- If **A**: proceed with the audit, but add a `--- Dependency Gaps ---` section at the end
  listing every place where missing context prevented complete assessment.
- If **B**: wait for the additional files, then audit all of them together as a multi-file review.

**If no external dependencies are detected:** proceed directly to the audit without
mentioning this check.
