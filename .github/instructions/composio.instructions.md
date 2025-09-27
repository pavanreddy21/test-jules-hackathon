---
description: "Composio integration patterns for external tool access in AI agents"
---
# Composio Integration Instructions

## Overview

**Composio provides middleware for AI agents to access 100+ external services** (Google Sheets, GitHub, Slack, etc.). This document covers authentication, tool loading, and LlamaIndex integration.

**Core files**: `agent/agent/agent.py` (`_load_composio_tools`), `agent/agent/sheets_integration.py`

## Dynamic Tool Loading

Load tools via LlamaIndex provider in `agent/agent/agent.py`:

```python
def _load_composio_tools() -> List[Any]:
    tool_ids_str = os.getenv("COMPOSIO_TOOL_IDS", "").strip()
    if not tool_ids_str:
        return []
    
    from composio import Composio
    from composio_llamaindex import LlamaIndexProvider
    
    user_id = os.getenv("COMPOSIO_USER_ID", "default")
    tool_ids = [t.strip() for t in tool_ids_str.split(",")]
    
    composio = Composio(provider=LlamaIndexProvider())
    tools = composio.tools.get(user_id=user_id, tools=tool_ids)
    return list(tools) if tools else []
```

**Patterns**: Lazy loading, graceful degradation (returns empty list on failure), user scoping.

### Tool ID Patterns

```bash
COMPOSIO_TOOL_IDS="GOOGLESHEETS_*"  # Wildcard
COMPOSIO_TOOL_IDS="GOOGLESHEETS_CREATE_GOOGLE_SHEET1,GOOGLESHEETS_BATCH_UPDATE"  # Specific
COMPOSIO_TOOL_IDS="GOOGLESHEETS_*,GITHUB_*,SLACK_SEND_MESSAGE"  # Multiple
```

Find slugs: Composio App Directory, docs, or `composio tools list` CLI.

## Authentication Management

**Flow**: Check connection → Initiate if needed → Wait for user → Use tools

```python
# 1. Check
result = composio.tools.execute(
    slug="COMPOSIO_CHECK_ACTIVE_CONNECTION",
    arguments={"toolkit": "GOOGLESHEETS"}
)

# 2. Initiate if not connected
result = composio.tools.execute(
    slug="COMPOSIO_INITIATE_CONNECTION",
    arguments={"toolkit": "GOOGLESHEETS"}
)
auth_url = result.get("data", {}).get("auth_url")

# 3. User completes OAuth, then agent proceeds
```

**User Scoping**: Use `COMPOSIO_USER_ID` env var ("default" for single-user, per-user IDs for multi-user).

## Google Sheets Integration

Client initialization in `sheets_integration.py`:
```python
def get_composio_client():
    from composio import Composio
    user_id = os.getenv("COMPOSIO_USER_ID", "default")
    return Composio(), user_id
```

### Tool Execution Pattern

```python
result = composio.tools.execute(
    user_id=user_id,
    slug="GOOGLESHEETS_GET_SPREADSHEET_INFO",
    arguments={"spreadsheet_id": sheet_id}
)

if not result or not result.get("successful"):
    return None

response_data = result.get("data", {}).get("response_data", {})
```

**Common result structure**: `{"successful": bool, "data": {"response_data": {...}}, "error": str}`

### Key Operations

**Create**: `GOOGLESHEETS_CREATE_GOOGLE_SHEET1` with `{"title": "..."}`  
**Read info**: `GOOGLESHEETS_GET_SPREADSHEET_INFO` with `{"spreadsheet_id": "..."}`  
**Read data**: `GOOGLESHEETS_BATCH_GET` with `{"spreadsheet_id": "...", "ranges": ["Sheet1!A:Z"]}`  
**Write data**: `GOOGLESHEETS_BATCH_UPDATE` with `{"spreadsheet_id", "sheet_name", "values": [[...]]}`  
**Delete rows**: `GOOGLESHEETS_DELETE_DIMENSION` with dimension range

## API Integration

**Backend** (`server.py`): FastAPI routes at `/sheets/create`, `/sync-to-sheets`  
**Frontend** (`src/app/api/sheets/*`): Next.js routes proxy to backend at `http://127.0.0.1:9000`

## Best Practices

### 1. Error Handling

**Always check for successful execution:**
```python
result = composio.tools.execute(
    user_id=user_id,
    slug="GOOGLESHEETS_BATCH_GET",
    arguments={...}
)

if not result or not result.get("successful"):
    error_msg = result.get("error", "Unknown error") if result else "No response"
    print(f"Tool execution failed: {error_msg}")
    return None

# Proceed with data extraction
data = result.get("data", {})
```

### 2. Graceful Degradation

**Handle missing dependencies:**
```python
def _load_composio_tools() -> List[Any]:
    try:
        from composio import Composio
        from composio_llamaindex import LlamaIndexProvider
    except Exception as e:
        print(f"Composio not available: {e}")
        return []  # App still works without Composio
    
    # ... tool loading
```

### 3. User Feedback

**Provide clear status messages:**
```python
# Agent system prompt includes:
"- Before using ANY Google Sheets functionality, ALWAYS first call "
"COMPOSIO_CHECK_ACTIVE_CONNECTION with user_id='default' and toolkit "
"id is GOOGLESHEETS to check if Google Sheets is connected.\n"
"- If the connection is NOT active, call COMPOSIO_INITIATE_CONNECTION "
"to start the authentication flow.\n"
"- After initiating connection, tell the user: 'Please complete the "
"Google Sheets authentication in your browser, then respond with "
"\"connected\" to proceed.'\n"
```

### 4. Rate Limit Management

**Batch operations when possible:**
```python
# ✅ Good - single batch update
result = composio.tools.execute(
    slug="GOOGLESHEETS_BATCH_UPDATE",
    arguments={"values": all_rows}  # Update all at once
)

# ❌ Bad - multiple individual updates
for row in rows:
    composio.tools.execute(
        slug="GOOGLESHEETS_UPDATE_CELL",
        arguments={"value": row}  # Rate limit risk
    )
```

### 5. Credential Security

**Never expose credentials in client code:**
```python
# ✅ Good - credentials on backend
composio = Composio()  # Reads from env

# ❌ Bad - credentials in frontend
const apiKey = "comp_xxx..."  # NEVER DO THIS
```

### 6. Logging and Debugging

**Add comprehensive logging:**
```python
print(f"Loading Composio tools: {tool_ids} for user: {user_id}")
print(f"Successfully loaded {len(tools)} tools")
print(f"Tool execution result: {result}")
print(f"Response data keys: {list(response_data.keys())}")
```

## Common Pitfalls

### 1. Missing Environment Variables

❌ **Wrong:** Assuming variables are set
```python
api_key = os.getenv("COMPOSIO_API_KEY")  # Might be None!
composio = Composio(api_key=api_key)
```

✅ **Correct:** Validate and provide defaults
```python
api_key = os.getenv("COMPOSIO_API_KEY")
if not api_key:
    print("COMPOSIO_API_KEY not set - Composio features disabled")
    return []

composio = Composio(api_key=api_key)
```

### 2. Incorrect Tool Slugs

❌ **Wrong:** Typo in tool slug
```python
result = composio.tools.execute(
    slug="GOOGLESHEETS_BATCH_UPDATES",  # Should be "BATCH_UPDATE"
    arguments={...}
)
```

✅ **Correct:** Verify slug names
```python
# Check available tools
tools = composio.tools.get(tools=["GOOGLESHEETS_*"])
print([tool.name for tool in tools])

# Use correct slug
result = composio.tools.execute(
    slug="GOOGLESHEETS_BATCH_UPDATE",
    arguments={...}
)
```

### 3. Ignoring Authentication Status

❌ **Wrong:** Calling tools without checking connection
```python
result = composio.tools.execute(
    slug="GOOGLESHEETS_BATCH_GET",
    arguments={...}
)
# Fails if user not authenticated!
```

✅ **Correct:** Check connection first
```python
# Check if connected
connection_check = composio.tools.execute(
    slug="COMPOSIO_CHECK_ACTIVE_CONNECTION",
    arguments={"toolkit": "GOOGLESHEETS"}
)

if not connection_check.get("successful"):
    # Initiate auth flow
    return {"error": "Please authenticate with Google Sheets"}

# Proceed with tool call
result = composio.tools.execute(...)
```

### 4. Not Handling Partial Failures

❌ **Wrong:** Assuming all operations succeed
```python
for item in items:
    composio.tools.execute(...)  # One failure stops everything
```

✅ **Correct:** Track successes and failures
```python
results = []
for item in items:
    result = composio.tools.execute(...)
    results.append({
        "item_id": item.id,
        "success": result.get("successful", False),
        "error": result.get("error")
    })

# Report summary
successful = sum(1 for r in results if r["success"])
failed = len(results) - successful
```

### 5. Response Data Structure Assumptions

❌ **Wrong:** Assuming fixed response structure
```python
sheet_id = result["data"]["response_data"]["spreadsheet_id"]
# Crashes if structure differs!
```

✅ **Correct:** Safe navigation with defaults
```python
sheet_id = (
    result.get("data", {})
    .get("response_data", {})
    .get("spreadsheet_id", "")
)

if not sheet_id:
    print("Failed to extract sheet ID from response")
```

## Debugging Tips

### 1. Enable Verbose Logging
```python
import logging
logging.basicConfig(level=logging.DEBUG)

# Composio will output detailed logs
```

### 2. Inspect Tool Arguments
```python
print(f"Executing tool: {slug}")
print(f"Arguments: {json.dumps(arguments, indent=2)}")

result = composio.tools.execute(
    user_id=user_id,
    slug=slug,
    arguments=arguments
)

print(f"Result: {json.dumps(result, indent=2)}")
```

### 3. Test Tools via CLI
```bash
# Install Composio CLI
pip install composio-core

# List available tools
composio tools list

# Test tool execution
composio tools execute GOOGLESHEETS_GET_SPREADSHEET_INFO \
  --args '{"spreadsheet_id": "your_sheet_id"}'
```

### 4. Verify Authentication
```bash
# Check active connections
composio connections list

# Check specific integration
composio connections show GOOGLESHEETS
```

### 5. Monitor API Usage
- Visit [Composio Dashboard](https://app.composio.dev/dashboard)
- Check "Logs" tab for tool execution history
- Review rate limits and quota usage

## Integration Checklist

When adding new Composio integrations:

- [ ] Environment variables configured in `agent/.env`
- [ ] Tool IDs added to `COMPOSIO_TOOL_IDS`
- [ ] Authentication flow handled (check connection → initiate → wait)
- [ ] Tool execution includes error handling
- [ ] Response data structure validated with safe navigation
- [ ] User feedback provided for auth and errors
- [ ] Rate limits considered for batch operations
- [ ] Backend API routes created (if needed)
- [ ] Frontend API routes created (if needed)
- [ ] Logging added for debugging
- [ ] Tested with actual API credentials

## Summary

Composio enables external tool integration through:
- **Dynamic tool loading** via LlamaIndex provider
- **Managed authentication** with user scoping
- **Standardized execution** via `composio.tools.execute()`
- **Google Sheets integration** for bidirectional sync
- **Graceful error handling** and degradation

Key principles: validate auth status, handle errors gracefully, batch operations, secure credentials server-side.