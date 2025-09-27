---
description: "CopilotKit integration patterns for frontend AI interactions"
---
# CopilotKit Integration Instructions

## Overview

**CopilotKit enables bidirectional React ↔ AI agent communication**. This document covers state synchronization, action definitions, and human-in-the-loop patterns.

**Core files**: `src/app/page.tsx`, `src/app/api/copilotkit/route.ts`

## Project-Specific CopilotKit Architecture

### Frontend Location
Located in `src/app/page.tsx`:
- Main component using `useCoAgent` for state synchronization
- Multiple `useCopilotAction` hooks exposing UI operations as tools
- `CopilotChat` component for desktop and `CopilotPopup` for mobile

### Runtime Configuration
Located in `src/app/api/copilotkit/route.ts`:

```typescript
import { CopilotRuntime } from "@copilotkit/runtime";
import { LlamaIndexAgent } from "@ag-ui/llamaindex";

const runtime = new CopilotRuntime({
  agents: {
    sample_agent: new LlamaIndexAgent({
      url: "http://127.0.0.1:9000/run",
    })
  }
});
```

**Key Configuration:**
- **Agent name**: `sample_agent` (must match `useCoAgent` name)
- **Backend URL**: `http://127.0.0.1:9000/run` (Python agent endpoint)
- **Adapter**: `LlamaIndexAgent` for AG-UI protocol translation
- **Endpoint**: `/api/copilotkit` (Next.js API route)

## Core Hooks & Components

### 1. useCoAgent - Bidirectional State Synchronization

**Purpose**: Synchronize application state between frontend and backend agent.

**Basic Usage:**
```typescript
import { useCoAgent } from "@copilotkit/react-core";

const { state, setState } = useCoAgent<AgentState>({
  name: "sample_agent",
  initialState,
});
```

**Type Definition:**
```typescript
type AgentState = {
  items: Item[];
  globalTitle: string;
  globalDescription: string;
  lastAction: string;
  itemsCreated: number;
  syncSheetId: string;
  syncSheetName: string;
};
```

**How It Works:**
1. **Initial State**: Frontend provides `initialState` to backend on first connection
2. **State Updates**: Backend receives full state snapshot on each agent invocation
3. **State Deltas**: Backend can send incremental updates (JSON Patch) or full snapshots
4. **UI Updates**: React automatically re-renders when state changes

**Critical Rules:**
- ✅ State must be JSON-serializable (no functions, DOM refs, etc.)
- ✅ Always use `setState` to update state, never mutate directly
- ✅ Backend receives **read-only** state - mutations via tools only
- ❌ Don't cache state in refs unless necessary (see State Caching Workaround)

**State Update Pattern:**
```typescript
setState((prev) => {
  const base = prev ?? initialState;
  const items = [...(base.items ?? []), newItem];
  return { ...base, items } as AgentState;
});
```

**Agent Name Matching:**
```typescript
// Frontend (page.tsx)
useCoAgent({ name: "sample_agent" })

// Backend route (route.ts)
new CopilotRuntime({
  agents: {
    sample_agent: new LlamaIndexAgent({...})  // MUST MATCH
  }
})
```

### 2. useCopilotAction - Exposing Frontend Tools

**Purpose**: Register functions that the agent can call to manipulate UI state.

**Basic Pattern:**
```typescript
import { useCopilotAction } from "@copilotkit/react-core";

useCopilotAction({
  name: "createItem",
  description: "Create a new canvas item and return its id.",
  available: "remote",  // Expose to backend agent
  parameters: [
    { 
      name: "type", 
      type: "string", 
      required: true, 
      description: "One of: project, entity, note, chart." 
    },
    { 
      name: "name", 
      type: "string", 
      required: false, 
      description: "Optional item name." 
    },
  ],
  handler: ({ type, name }: { type: CardType; name?: string }) => {
    const itemId = addItem(type, name);
    return { itemId };  // Return value sent back to agent
  },
});
```

**Parameter Schema:**
- `name` - Tool name visible to agent (must match Python stub in `agent.py`)
- `description` - Tool description shown to LLM
- `available: "remote"` - Makes tool callable by backend agent
- `parameters` - Array of parameter definitions with types and descriptions
- `handler` - Function executed when agent calls the tool

**Parameter Types:**
- `"string"`, `"number"`, `"boolean"`, `"object"`, `"array"`
- Use `required: true/false` for mandatory parameters
- Descriptions guide the LLM on parameter usage

**Return Values:**
- Return an object to send data back to agent
- Return `void` if no data needed
- Agent sees return value in tool execution result

**Example with State Update:**
```typescript
useCopilotAction({
  name: "setProjectField1",
  description: "Update project.data.field1 (text field).",
  available: "remote",
  parameters: [
    { name: "value", type: "string", required: true },
    { name: "itemId", type: "string", required: true },
  ],
  handler: ({ value, itemId }) => {
    updateItemData(itemId, (prev) => {
      const pd = prev as ProjectData;
      return { ...pd, field1: value };
    });
  },
});
```

### renderAndWaitForResponse - Human-in-the-Loop (HITL)

Pause agent to ask user for input:

```typescript
useCopilotAction({
  name: "choose_item",
  renderAndWaitForResponse: ({ respond, args }) => (
    <div>
      <p>{args.prompt}</p>
      {args.choices.map((c) => (
        <button key={c} onClick={() => respond(c)}>{c}</button>
      ))}
    </div>
  ),
});
```

**Flow**: Agent calls → UI renders → User responds via `respond()` → Agent continues.

**Use cases**: Disambiguation, confirmation, user choice when agent can't decide.

### useCopilotAdditionalInstructions

Inject dynamic context into agent's system prompt:

```typescript
useCopilotAdditionalInstructions({
  instructions: (() => {
    const items = state.items.slice(0, 5);
    return `Current canvas has ${state.items.length} items. Recent: ${items.map(i => i.name).join(", ")}`;
  })()
});
```

**Use**: State snapshots, contextual constraints, user preferences. Keep lightweight - runs every render.

## UI Components

**CopilotChat**: Full chat panel  
**CopilotPopup**: Floating button  
**CopilotSidebar**: Side panel

```typescript
import { CopilotPopup } from "@copilotkit/react-ui";
import "@copilotkit/react-ui/styles.css";

<CopilotPopup labels={{ title: "Canvas Assistant" }} />
```
const isDesktop = useMediaQuery("(min-width: 768px)");

{isDesktop ? (
  <CopilotChat {...chatProps} />
) : (
  <CopilotPopup {...chatProps} />
)}
```

## Advanced Patterns

### State Caching Workaround

**Problem**: CopilotKit sometimes sends transient empty state during updates, causing flicker.

**Solution**: Cache last valid state and use it during transitions.

```typescript
## Advanced Patterns

### State Caching Workaround

CopilotKit can send transient empty states during sync. Cache the last valid state:

```typescript
const cachedStateRef = useRef(initialState);
const viewState = isNonEmptyAgentState(state) 
  ? (state as AgentState) 
  : cachedStateRef.current;
```

Temporary fix - report underlying issue to CopilotKit if encountered.

### Deduplication Patterns

Prevent AI agent loops from creating duplicates:

**Time-based throttling** (5-second window):
```typescript
const lastCreationRef = useRef<{type, name, id, ts}>(null);

if (last && last.type === type && last.name === name && now - last.ts < 5000) {
  return { itemId: last.id };  // Return existing
}
```

**Name-based**: Check `state.items.find(i => i.type === type && i.name === name)` before creating.

**Sub-item**: Check existing checklist/metrics before adding.
```

**Helper Function:**
```typescript
function isNonEmptyAgentState(s: unknown): boolean {
  return s != null 
    && typeof s === "object" 
    && Array.isArray((s as AgentState).items);
}
```

**When to Use:**
- Rendering lists/grids that flicker on updates
- Critical UI elements that shouldn't disappear
- As temporary fix - report underlying issue to CopilotKit

### Deduplication Patterns

**Problem**: AI agents can create duplicate items in loops.

**Solution**: Track recent creations and reject duplicates.

**Pattern 1: Time-based throttling**
```typescript
const lastCreationRef = useRef<{
  type: CardType;
  name: string;
  id: string;
  ts: number;
} | null>(null);

useCopilotAction({
  name: "createItem",
  handler: ({ type, name }) => {
    // Check for duplicate creation within 5 seconds
    const now = Date.now();
    const last = lastCreationRef.current;
    
    if (last && 
        last.type === type && 
        last.name === name && 
        now - last.ts < 5000) {
      return { itemId: last.id };  // Return existing
    }
    
    // Create new item
    const itemId = addItem(type, name);
    lastCreationRef.current = { type, name, id: itemId, ts: now };
    return { itemId };
  },
});
```

**Pattern 2: Name-based deduplication**
```typescript
useCopilotAction({
  name: "createItem",
  handler: ({ type, name }) => {
    const base = state ?? initialState;
    const items = base.items ?? [];
    
    // Check for existing item with same type+name
    const existing = items.find(
      (item) => item.type === type && item.name === name
    );
    
    if (existing) {
      return { itemId: existing.id };  // Return existing
    }
    
    // Create new
    const itemId = addItem(type, name);
    return { itemId };
  },
});
```

**Sub-item**: Check existing checklist/metrics before adding.

### Date Normalization

Normalize all dates to `YYYY-MM-DD`:

```typescript
function normalizeDate(dateInput: string | Date): string {
  if (typeof dateInput === "string" && /^\d{4}-\d{2}-\d{2}$/.test(dateInput)) {
    return dateInput;
  }
  const date = typeof dateInput === "string" ? new Date(dateInput) : dateInput;
  return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, "0")}-${String(date.getDate()).padStart(2, "0")}`;
}
```

Handles natural language ("tomorrow"), ISO strings, Date objects.

### Complex State Updates

Use updater functions from `@/lib/canvas/updates` for nested updates:

```typescript
useCopilotAction({
  name: "setProjectChecklistItem",
  handler: ({ itemId, checklistItemId, text, done }) => {
    updateItemData(itemId, (prev) => {
      let pd = prev as ProjectData;
      if (text) pd = projectSetField4ItemText(pd, checklistItemId, text);
      if (done !== undefined) pd = projectSetField4ItemDone(pd, checklistItemId, done);
      return pd;
    });
  },
});
```

Helper functions in `src/lib/canvas/updates.ts` handle immutable nested updates.

## Best Practices

### Action Naming
- ✅ `createItem`, `setItemName`, `set{Type}Field{Number}`
- ❌ Generic names like `update`, `change`

### Parameter Descriptions
Be specific about format and purpose:
```typescript
{ name: "date", type: "string", description: "YYYY-MM-DD format for project.data.field3" }
```

### Error Handling
Validate inputs and handle edge cases. Return meaningful results (e.g., existed flag) for confirmation.

### Return Values
Return data the agent needs - created IDs, success flags, validation messages.

### Optimization
Memoize callbacks and computations using `useCallback`, `useMemo` to prevent re-renders.

### Type Safety
Use TypeScript types for all handler parameters and return values.

## Common Pitfalls

### Agent Name Mismatch
Ensure agent name matches between `useCoAgent({ name: "sample_agent" })` and CopilotKit route configuration.

### Missing `available: "remote"`
Always include `available: "remote"` in `useCopilotAction` to expose to agent.

### Non-Serializable State
### Non-Serializable State
Only use plain data (strings, numbers, objects, arrays) - no functions, DOM refs, or class instances.

### Direct State Mutation
Always use `setState` with updater function - never mutate state directly.

### Undefined State Handling
Always provide fallback to `initialState` when accessing state properties.

## Debugging Tips

- **"Agent can't see tool"**: Check `available: "remote"` and agent name matching
- **Empty state on render**: Use `cachedStateRef` pattern to prevent flicker
- **Infinite loops**: Check deduplication logic and `lastAction` tracking
- **Type errors**: Ensure frontend types match backend `FIELD_SCHEMA`
- **Monitor state**: Use `useEffect` to log state changes
- **Log actions**: Add console.log in handlers to trace execution

## Advanced Patterns

### Auto-sync to External Systems

Use `useEffect` with debouncing:

```typescript
useEffect(() => {
  const timer = setTimeout(() => {
    if (state?.items) {
      fetch("/api/sheets/sync", { method: "POST", body: JSON.stringify(state) });
    }
  }, 2000);
  return () => clearTimeout(timer);
}, [state]);
```

### Multi-step Workflows

Chain actions by returning IDs:

```typescript
const { itemId } = await createItem({ type: "project" });
await setProjectField1({ itemId, text: "description" });
```

### Conditional Tool Availability

```typescript
useCopilotAction({
  name: "deleteItem",
  available: ({ state }) => (state?.items?.length ?? 0) > 0,
  handler: ({ itemId }) => { /* ... */ },
});
```

## Integration Checklist

- [ ] Action name matches Python stub in `agent.py`
- [ ] `available: "remote"` set
- [ ] Agent name matches between `useCoAgent` and runtime config
- [ ] Parameter descriptions clear
- [ ] Error handling in place
- [ ] Immutable state updates
- [ ] Return useful values to agent
- [ ] Types match frontend/backend

## Summary

CopilotKit bridges React UI and AI agents through `useCoAgent`, `useCopilotAction`, `renderAndWaitForResponse`, and the `LlamaIndexAgent` adapter for AG-UI protocol translation.
