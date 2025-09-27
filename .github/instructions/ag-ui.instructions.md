---
description: "AG-UI (Agentic UI) protocol guidelines and patterns"
---
# AG-UI (Agentic UI) Protocol Instructions

## Overview

**AG-UI is an event-driven protocol that standardizes communication between backend AI agents and frontend applications**. This document focuses on AG-UI protocol-specific patterns in this project. For project architecture and implementation details, see the main Copilot instructions.

## GitHub Repository Reference

When working with AG-UI concepts, search the official GitHub repository (`githubRepo` tool):

**Repository**: `ag-ui-protocol/ag-ui`

## Backend AG-UI Router Configuration

Located in `agent/agent.py`:

### Tool Placement Strategy

**Frontend tools** (`frontend_tools`):
- UI manipulation, visual feedback
- Safe client-side operations
- Examples: Canvas CRUD operations, HITL actions

**Backend tools** (`backend_tools`):
- External APIs with secrets
- Heavy computation, database operations
- Examples: Composio integrations (Google Sheets)

## AG-UI Event Types

**For complete reference**, search `ag-ui-protocol/ag-ui` at `/docs/concepts/events.mdx`.

### Events Used in This Project

- `TEXT_MESSAGE_CONTENT` - Agent chat responses
- `TOOL_CALL_START` / `TOOL_CALL_END` - Tool execution lifecycle
- `STATE_DELTA` - State synchronization (JSON Patch RFC 6902)
- `STATE_SNAPSHOT` - Complete state at a point in time
- Lifecycle: `RUN_STARTED`, `RUN_FINISHED`, `RUN_ERROR`

### State Delta Pattern

AG-UI uses **incremental updates** to prevent unnecessary re-renders:
- `STATE_SNAPSHOT` sends complete state
- `STATE_DELTA` sends JSON Patch operations for granular changes

Frontend automatically merges deltas.

## Frontend Integration via CopilotKit

Located in `src/app/api/copilotkit/route.ts`:

```typescript
import { LlamaIndexAgent } from "@copilotkit/sdk-js/llamaindex";

const agent = new LlamaIndexAgent({
  name: "sample_agent",
  url: "http://127.0.0.1:9000",
});
```

The `LlamaIndexAgent` adapter:
- Connects to Python backend
- Translates AG-UI protocol
- Manages WebSocket/SSE connection
- Forwards events to React components

**Critical**: Agent name must match `useCoAgent({ name: "sample_agent" })` in frontend.

## Debugging AG-UI Events

**Monitor raw protocol messages**:
1. Enable verbose logging: `pnpm dev:debug` (sets `LOG_LEVEL=debug`)
2. Check network tab for WebSocket/SSE events
3. Look for AG-UI event types in message payloads

**Common connection issues**:
- Backend not running on port 9000
- Agent name mismatch between route.ts and page.tsx
- Missing `OPENAI_API_KEY` in `agent/.env`