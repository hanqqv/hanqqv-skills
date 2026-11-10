# unslop-technical-writing

Removes AI slop from technical writing: READMEs and `.md` docs, design docs, code comments, godoc and docstrings, commit messages, PR descriptions and Jira text.

**Triggers on:** writing or editing any of the above · "unslop this" · "too wordy" · "sounds like ChatGPT" · `/unslop-technical-writing`

**What it does:**

- Checks facts first. A claim stays only if code, a spec, command output or the user's own text backs it. It doesn't fill gaps with likely-sounding details such as status codes, field names or "tests pass".
- States each fact once instead of repeating a limit in the table, the steps and the example.
- Keeps comments that explain why and deletes ones that repeat the next line. It doesn't invent a reason it can't source.
- Gives a pattern → fix table for common slop: throat-clearing openers, hype words, hedging, "In summary" endings, emoji and "Key Features" lists, notes to the reader, "it's not X, it's Y".
- In editing mode, keeps every fact in the draft and flags doubtful ones in its reply instead of deleting them.
