<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>StackEdit — Undo/Redo with Stacks</title>
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body {
    font-family: 'Segoe UI', system-ui, sans-serif;
    background: #0f172a;
    color: #e2e8f0;
    min-height: 100vh;
    padding: 24px;
  }
  header { text-align: center; margin-bottom: 24px; }
  header h1 { font-size: 1.8rem; color: #38bdf8; }
  header p { color: #94a3b8; font-size: 0.9rem; margin-top: 4px; }

  .layout {
    display: grid;
    grid-template-columns: 2fr 1fr;
    gap: 20px;
    max-width: 1100px;
    margin: 0 auto;
  }
  @media (max-width: 800px) { .layout { grid-template-columns: 1fr; } }

  .panel {
    background: #1e293b;
    border: 1px solid #334155;
    border-radius: 12px;
    padding: 16px;
  }
  .panel h2 {
    font-size: 0.85rem;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    color: #94a3b8;
    margin-bottom: 12px;
  }

  textarea {
    width: 100%;
    height: 260px;
    background: #0f172a;
    color: #e2e8f0;
    border: 1px solid #334155;
    border-radius: 8px;
    padding: 12px;
    font-family: 'Consolas', monospace;
    font-size: 1rem;
    resize: vertical;
    outline: none;
  }
  textarea:focus { border-color: #38bdf8; }

  .toolbar { display: flex; gap: 10px; margin-top: 12px; flex-wrap: wrap; }
  button {
    background: #38bdf8;
    color: #0f172a;
    border: none;
    padding: 10px 18px;
    border-radius: 8px;
    font-weight: 600;
    cursor: pointer;
    transition: 0.15s;
    font-size: 0.9rem;
  }
  button:hover:not(:disabled) { background: #7dd3fc; }
  button:disabled { background: #334155; color: #64748b; cursor: not-allowed; }
  button.secondary { background: #475569; color: #e2e8f0; }
  button.secondary:hover:not(:disabled) { background: #64748b; }

  .stack-col { margin-bottom: 20px; }
  .stack-box {
    min-height: 90px;
    border: 2px dashed #334155;
    border-radius: 8px;
    padding: 8px;
    display: flex;
    flex-direction: column-reverse; /* top of stack renders at top */
    gap: 6px;
  }
  .stack-item {
    background: #0ea5e9;
    color: #0f172a;
    padding: 6px 10px;
    border-radius: 6px;
    font-family: monospace;
    font-size: 0.8rem;
    animation: pop 0.2s ease;
  }
  .redo .stack-item { background: #a78bfa; }
  .stack-item.top { outline: 2px solid #facc15; }
  .empty-hint {
    color: #475569;
    font-size: 0.8rem;
    text-align: center;
    padding: 20px 0;
    font-style: italic;
  }
  @keyframes pop {
    from { transform: scale(0.85); opacity: 0; }
    to { transform: scale(1); opacity: 1; }
  }

  .status {
    margin-top: 12px;
    font-size: 0.85rem;
    color: #94a3b8;
    min-height: 20px;
  }
  .status b { color: #38bdf8; }
</style>
</head>
<body>

<header>
  <h1>📚 StackEdit</h1>
  <p>A text editor powered by two LIFO stacks — watch them grow and shrink as you type</p>
</header>

<div class="layout">
  <!-- EDITOR -->
  <div class="panel">
    <h2>Editor</h2>
    <textarea id="editor" placeholder="Start typing here... every keystroke is pushed onto the undo stack."></textarea>
    <div class="toolbar">
      <button id="undoBtn" class="secondary" disabled>↶ Undo (Ctrl+Z)</button>
      <button id="redoBtn" class="secondary" disabled>↷ Redo (Ctrl+Y)</button>
      <button id="clearBtn">Clear All</button>
    </div>
    <div class="status" id="status">Ready. Start typing.</div>
  </div>

  <!-- STACKS -->
  <div class="panel">
    <div class="stack-col">
      <h2>Undo Stack (top = next to undo)</h2>
      <div class="stack-box" id="undoStack"></div>
    </div>
    <div class="stack-col">
      <h2>Redo Stack (top = next to redo)</h2>
      <div class="stack-box redo" id="redoStack"></div>
    </div>
  </div>
</div>

<script>
/**
 * Stack-based undo/redo engine.
 *
 * We track every mutation as an Action:
 *   { type: "INSERT" | "DELETE" | "CLEAR", char, position, prevText }
 *
 * - On any edit: push action to undoStack, clear redoStack.
 * - Undo: pop from undoStack, reverse the action, push to redoStack.
 * - Redo: pop from redoStack, re-apply action, push to undoStack.
 */

const editor   = document.getElementById('editor');
const undoBtn  = document.getElementById('undoBtn');
const redoBtn  = document.getElementById('redoBtn');
const clearBtn = document.getElementById('clearBtn');
const undoBox  = document.getElementById('undoStack');
const redoBox  = document.getElementById('redoStack');
const statusEl = document.getElementById('status');

const undoStack = [];
const redoStack = [];

let lastLength = 0; // to detect whether a change was an insert or delete

/* ---------- Core stack logic ---------- */

function pushAction(action) {
  undoStack.push(action);
  redoStack.length = 0;         // any new edit invalidates redo history
  render();
}

function undo() {
  if (undoStack.length === 0) return;
  const action = undoStack.pop();

  // Reverse the action on the text
  if (action.type === 'INSERT') {
    // remove the inserted char
    editor.value =
      action.prevText; // simplest reliable reversal
  } else if (action.type === 'DELETE') {
    editor.value = action.prevText;
  } else if (action.type === 'CLEAR') {
    editor.value = action.prevText;
  }

  redoStack.push(action);
  lastLength = editor.value.length;
  render();
  setStatus(`Undid <b>${action.type}</b> of "${action.char || ''}"`);
}

function redo() {
  if (redoStack.length === 0) return;
  const action = redoStack.pop();

  if (action.type === 'INSERT') {
    editor.value = action.newText;
  } else if (action.type === 'DELETE') {
    editor.value = action.newText;
  } else if (action.type === 'CLEAR') {
    editor.value = '';
  }

  undoStack.push(action);
  lastLength = editor.value.length;
  render();
  setStatus(`Redid <b>${action.type}</b> of "${action.char || ''}"`);
}

/* ---------- Detect edits from the textarea ---------- */

editor.addEventListener('input', () => {
  const newText = editor.value;
  const newLen = newText.length;
  const prevText = lastText;

  if (newLen > lastLength) {
    // INSERT — find the inserted character(s)
    const pos = findChangeIndex(prevText, newText);
    const char = newText.slice(pos, pos + (newLen - lastLength));
    pushAction({
      type: 'INSERT',
      char,
      position: pos,
      prevText,
      newText,
    });
    setStatus(`Pushed <b>INSERT</b> "${char}" @ ${pos}`);
  } else if (newLen < lastLength) {
    // DELETE
    const pos = findChangeIndex(prevText, newText);
    const char = prevText.slice(pos, pos + (lastLength - newLen));
    pushAction({
      type: 'DELETE',
      char,
      position: pos,
      prevText,
      newText,
    });
    setStatus(`Pushed <b>DELETE</b> "${char}" @ ${pos}`);
  }

  lastText = newText;
  lastLength = newLen;
  render();
});

// Track the previous text snapshot
let lastText = '';

function findChangeIndex(a, b) {
  let i = 0;
  while (i < a.length && i < b.length && a[i] === b[i]) i++;
  return i;
}

/* ---------- Clear ---------- */

clearBtn.addEventListener('click', () => {
  if (editor.value === '') return;
  pushAction({
    type: 'CLEAR',
    char: '',
    position: 0,
    prevText: editor.value,
    newText: '',
  });
  editor.value = '';
  lastText = '';
  lastLength = 0;
  setStatus('Pushed <b>CLEAR</b>');
  render();
});

/* ---------- Buttons ---------- */

undoBtn.addEventListener('click', undo);
redoBtn.addEventListener('click', redo);

// Keyboard shortcuts
document.addEventListener('keydown', (e) => {
  if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === 'z' && !e.shiftKey) {
    e.preventDefault();
    undo();
  } else if (
    (e.ctrlKey || e.metaKey) &&
    (e.key.toLowerCase() === 'y' || (e.shiftKey && e.key.toLowerCase() === 'z'))
  ) {
    e.preventDefault();
    redo();
  }
});

/* ---------- Rendering ---------- */

function render() {
  undoBtn.disabled = undoStack.length === 0;
  redoBtn.disabled = redoStack.length === 0;
  renderStack(undoBox, undoStack, 'undo stack empty');
  renderStack(redoBox, redoStack, 'redo stack empty');
}

function renderStack(box, stack, emptyMsg) {
  box.innerHTML = '';
  if (stack.length === 0) {
    box.innerHTML = `<div class="empty-hint">${emptyMsg}</div>`;
    return;
  }
  // Show top of stack first (visually at the top)
  [...stack].reverse().forEach((action, idx) => {
    const div = document.createElement('div');
    div.className = 'stack-item' + (idx === 0 ? ' top' : '');
    const label = action.char
      ? `${action.type}("${action.char}") @${action.position}`
      : `${action.type}()`;
    div.textContent = label;
    box.appendChild(div);
  });
}

function setStatus(html) {
  statusEl.innerHTML = html;
}

/* ---------- Init ---------- */

lastLength = 0;
lastText = '';
render();
</script>
</body>
</html>
