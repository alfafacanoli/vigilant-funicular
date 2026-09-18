# wc + context protocol

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Production Web Components Solution</title>
<style>
body {
font-family: system-ui, -apple-system, sans-serif;
max-width: 500px;
margin: 40px auto;
padding: 0 20px;
background: #fdfdfd;
color: #222;
}
</style>
</head>
<body>

<!-- The unified app shell component -->
<task-manager></task-manager>

<script type="module">
// ==========================================
// 1. ENGINE ENGINE CORNER (Store.js Core)
// ==========================================
class GlobalStore {
#state = {};
#listeners = new Set();
#selectorsCache = new WeakMap();

registerModule(symbolToken, initialModuleState) {
if (typeof symbolToken !== 'symbol') throw new TypeError('Requires Symbol token.');
if (this.#state[symbolToken]) throw new Error('Already registered.');
this.#state[symbolToken] = initialModuleState;
}

select(selectorFn) {
if (this.#selectorsCache.has(selectorFn)) {
return this.#selectorsCache.get(selectorFn);
}

let lastSlice = selectorFn(this.#state);
const moduleCallbacks = new Set();

const internalListener = (nextState) => {
const nextSlice = selectorFn(nextState);
if (this.#isEqual(lastSlice, nextSlice)) return;

lastSlice = nextSlice;
const frozenSlice = Object.freeze(nextSlice);

for (const cb of moduleCallbacks) cb(frozenSlice);
};

const selection = {
get value() { return lastSlice; },
onChange: (callback) => {
if (moduleCallbacks.size === 0) this.#listeners.add(internalListener);
moduleCallbacks.add(callback);

// RENDERING FIX: Immediately fire the callback once with the current
// state slice so the UI populates right away upon subscription!
callback(Object.freeze(lastSlice));

return () => {
moduleCallbacks.delete(callback);
if (moduleCallbacks.size === 0) this.#listeners.delete(internalListener);
};
}
};

this.#selectorsCache.set(selectorFn, selection);
return selection;
}

update(symbolToken, recipe) {
const draft = structuredClone(this.#state[symbolToken]);
try {
recipe(draft);
this.#state[symbolToken] = draft;
this.#notify();
} catch (error) {
console.error(`State update rejected:`, error);
}
}

#notify() {
for (const listener of this.#listeners) listener(this.#state);
}

#isEqual(a, b) {
if (a === b) return true;
if (typeof a !== 'object' || typeof b !== 'object' || a === null || b === null) return false;
const keysA = Reflect.ownKeys(a);
const keysB = Reflect.ownKeys(b);
if (keysA.length !== keysB.length) return false;
return keysA.every(key => a[key] === b[key]);
}
}

const globalStore = new GlobalStore();

// ==========================================
// 2. FEATURE-SCOPED CONTEXT TOKEN KEYS
// ==========================================
class ContextKey {
#description;
constructor(description) { this.#description = description; }
}
const TasksListContext = new ContextKey('tasks-data-stream');

// Initialize isolated state sandbox
const TASKS_TOKEN = Symbol('tasks-module-slice');
globalStore.registerModule(TASKS_TOKEN, {
items: ['Master Stringless Contexts', 'Build a Bulletproof Unidirectional Loop']
});

const selectTaskItems = (state) => state[TASKS_TOKEN]?.items || [];
const liveTasksSelection = globalStore.select(selectTaskItems);

// ==========================================
// 3. SMART CONTAINER SHELL (task-manager)
// ==========================================
class TaskManager extends HTMLElement {
#contextProviders = new Map([
[TasksListContext, liveTasksSelection]
]);

constructor() {
super();
this.attachShadow({ mode: 'open' });
this.shadowRoot.innerHTML = `
<h1>Task Dashboard (Smart Shell)</h1>
<task-input></task-input>
<hr>
<task-list></task-list>
`;
}

connectedCallback() {
this.shadowRoot.addEventListener('context-request', (e) => {
const { context, callback } = e.detail;
if (this.#contextProviders.has(context)) {
e.stopPropagation();
callback(this.#contextProviders.get(context));
}
});

this.shadowRoot.addEventListener('task-added', (e) => {
globalStore.update(TASKS_TOKEN, (draft) => { draft.items.push(e.detail); });
});

this.shadowRoot.addEventListener('task-deleted', (e) => {
globalStore.update(TASKS_TOKEN, (draft) => { draft.items.splice(e.detail, 1); });
});
}
}
customElements.define('task-manager', TaskManager);

// ==========================================
// 4. INPUT LEAF COMPONENT (task-input)
// ==========================================
class TaskInput extends HTMLElement {
constructor() {
super();
this.attachShadow({ mode: 'open' });
this.shadowRoot.innerHTML = `
<form id="task-form">
<input type="text" id="input" placeholder="Enter a new task..." required />
<button type="submit">Add Task</button>
</form>
`;
}

connectedCallback() {
this.shadowRoot.getElementById('task-form').addEventListener('submit', (e) => {
e.preventDefault();
const input = this.shadowRoot.getElementById('input');
this.dispatchEvent(new CustomEvent('task-added', {
detail: input.value,
bubbles: true,
composed: true
}));
input.value = '';
});
}
}
customElements.define('task-input', TaskInput);

// ==========================================
// 5. SELF-BINDING LIST LEAF (task-list)
// ==========================================
class TaskList extends HTMLElement {
#unsub = null;
#tasks = [];

constructor() {
super();
this.attachShadow({ mode: 'open' });
this.shadowRoot.innerHTML = `<ul id="list-container"></ul>`;
}

connectedCallback() {
this.shadowRoot.getElementById('list-container').addEventListener('click', (e) => {
const deleteBtn = e.target.closest('.delete-btn');
if (deleteBtn) {
const index = parseInt(deleteBtn.dataset.index, 10);
this.dispatchEvent(new CustomEvent('task-deleted', {
detail: index,
bubbles: true,
composed: true
}));
}
});

// Delay event slightly into the microtask queue
// to let the parent task-manager setup its listeners first.
queueMicrotask(() => {
this.dispatchEvent(new CustomEvent('context-request', {
bubbles: true,
composed: true,
detail: {
context: TasksListContext,
callback: (liveSelectionStream) => {
this.#unsub = liveSelectionStream.onChange((latestTasks) => {
this.#tasks = latestTasks;
this.render();
});
}
}
}));
});
}

render() {
const container = this.shadowRoot.getElementById('list-container');
if (this.#tasks.length === 0) {
container.innerHTML = `<li>No tasks yet!</li>`;
return;
}
container.innerHTML = this.#tasks
.map((task, index) => `<li><span>${task}</span> <button class="delete-btn" data-index="${index}">❌</button></li>`)
.join('');
}

disconnectedCallback() {
if (this.#unsub) {
this.#unsub();
this.#unsub = null;
}
}
}
customElements.define('task-list', TaskList);
</script>
</body>
</html>




Sent from Proton Mail for Android. 
```
