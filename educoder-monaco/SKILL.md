---
name: educoder-monaco
description: >-
  This skill is for interacting with Monaco code editors on the EduCoder (TouGe/头歌) platform via Playwright browser automation.
  Use when the user needs to read or modify code in EduCoder's online IDE.
  The skill covers reading editor content, modifying code via Monaco model API, and submitting evaluations.
agent_created: true
---

# EduCoder Monaco Editor Operations

This skill provides the workflow for operating Monaco code editors on the EduCoder (头歌实践教学平台) via playwright-cli browser automation.

## Key Architecture

EduCoder uses Monaco Editor wrapped in React. The editor has two layers:
1. **Monaco Models** (via `window.Monaco.editor.getModels()`) — the actual content
2. **React State** (via `window.updateMonacoValue()`) — a visual sync helper

**Critical distinction**: `window.updateMonacoValue()` only updates React state and the visual display. It does NOT update the Monaco model. When you click "评测" (evaluate), the platform reads from the Monaco model, not from React state.

## Reading Editor Content

Get the current content from the Monaco model:

```bash
playwright-cli run-code "async page => { const content = await page.evaluate(() => { const models = window.Monaco?.editor?.getModels ? window.Monaco.editor.getModels() : []; return models.length > 0 ? models[0].getValue() : 'no models'; }); return content; }"
```

To list all models with their contents:

```bash
playwright-cli run-code "async page => { return await page.evaluate(() => { const models = window.Monaco.editor.getModels(); return models.map((m, i) => ({ i, uri: m.uri?.toString(), preview: m.getValue().substring(0, 80) })); }); }"
```

## Modifying Editor Content

**DO NOT use these approaches** (they will NOT work):
- `playwright-cli fill` on the textarea — Monaco intercepts input events differently
- `page.keyboard.insertText()` — the text is typed but Monaco drops it
- `Ctrl+A` + paste — Monaco handles paste differently
- `window.updateMonacoValue()` — only updates React state, NOT the model

**The CORRECT approach**: Directly modify the Monaco model via `setValue()`.

### Step 1: Write the modification script to a file

Create a JS file with the correct code (use template literals to avoid escaping issues):

```js
async (page) => {
  const newCode = `package step3;

public class QueryRecord {
    // Your corrected code here
}`;
  
  return await page.evaluate((code) => {
    const models = window.Monaco.editor.getModels();
    for (const m of models) {
      const v = m.getValue();
      // Match the model containing the file to edit
      if (v.includes('QueryRecord') && v.includes('WHERE bname')) {
        m.setValue(code);
        return 'updated: found by content match';
      }
    }
    // Fallback to first model
    if (models.length > 0) {
      models[0].setValue(code);
      return 'updated: fallback to model[0]';
    }
    return 'no models found';
  }, newCode);
}
```

### Step 2: Execute the script

```bash
CODE=$(cat "/path/to/script.js") && playwright-cli run-code "$CODE"
```

### Step 3: Sync React state (recommended)

After updating the model, sync the React visual state:

```bash
playwright-cli run-code "async page => { const code = await page.evaluate(() => window.Monaco.editor.getModels()[0].getValue()); await page.evaluate((c) => window.updateMonacoValue(c), code); return 'synced'; }"
```

## Clicking the Evaluate Button

```bash
playwright-cli run-code "async page => { const btn = page.getByRole('button', { name: /评测/ }); await btn.click(); return 'clicked'; }"
```

Then wait ~8 seconds for evaluation to complete and check results:

```bash
playwright-cli run-code "async page => { await page.waitForTimeout(8000); return await page.evaluate(() => { const passText = document.body.innerText; return passText.includes('全部通过') ? 'ALL PASSED' : passText.substring(passText.indexOf('测试集'), 400); }); }"
```

## Important Notes

- The `window.updateMonacoValue` function has signature `(value) => { setter(value) }` — it's a React state setter wrapper
- The Monaco editor container has `data-keybindingcontext` and `data-modeid` attributes
- Multiple Monaco models may exist (one per open file tab); model[0] is typically the active file
- Use file-based JS scripts (not inline bash) when the code contains single quotes to avoid shell escaping issues
