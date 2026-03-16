---
name: servicenow-workspace-builder
description: Build custom ServiceNow Next Experience workspaces, widgets (macroponents), pages, and routes programmatically via REST API and GlideRecord. Use this skill whenever the user asks to create, modify, or wire up a custom workspace in ServiceNow, create a macroponent or custom component for a workspace, add a widget/page to a workspace, troubleshoot workspace routing or rendering issues, or work with sys_ux tables (sys_ux_macroponent, sys_ux_page_registry, sys_ux_app_config, sys_ux_app_route, sys_ux_screen_type, sys_ux_screen). Also trigger when the user mentions UI Builder, Next Experience, Breadcrumb App Shell, workspace page wiring, or screen_type/macroponent linking. This skill covers the full record chain needed to make a workspace render correctly, which is NOT obvious and requires specific knowledge of 6+ linked tables.
---

# ServiceNow Workspace Builder

Build custom Next Experience workspaces programmatically via the ServiceNow REST API and server-side GlideRecord scripts.

## When to Use

- Creating a custom workspace from scratch
- Adding a custom widget (macroponent) to a workspace
- Wiring pages, routes, and screen types
- Troubleshooting "workspace loads but content is blank" issues
- Any work involving `sys_ux_*` tables

## Prerequisites

- ServiceNow instance URL and admin credentials
- Access to the REST Table API (`/api/now/table/`)
- Ability to run `curl` commands (via MCP server, bash, or terminal)

## Key Concept: The Workspace Record Chain

A workspace is NOT a single record. It requires 6+ linked records to render properly. This is the chain (read `references/record-chain.md` for full details):

```
sys_ux_page_registry (the app/workspace entry)
  -> root_macroponent = "Breadcrumb App Shell" (the chrome/shell)
  -> sys_ux_app_config (links registry to configuration)
  -> sys_ux_app_route (defines URL paths like /home)
       -> sys_ux_screen_type (defines the screen for the route)
            -> sys_ux_screen (CRITICAL: glue between screen_type and macroponent)
                 -> sys_ux_macroponent (your custom widget)
```

Missing ANY link in this chain = blank page inside the workspace shell.

## Workflow

### Step 1: Create the Macroponent (Widget)

POST to `sys_ux_macroponent`. Use a Stylized Text child component with a transform script for simple HTML rendering. See `references/record-chain.md` for the composition JSON structure.

### Step 2: Create the Page Registry (Workspace App)

POST to `sys_ux_page_registry` with:
- `title`, `path` (URL slug), `description`
- `root_macroponent` = sys_id of "Breadcrumb App Shell"
- `parent_app` and `admin_panel` (copy from a working workspace like CMDB)

### Step 3: Create Supporting Records via Scheduled Job

Some `sys_ux_*` tables block direct REST API inserts due to ACLs. The workaround is to create a one-shot scheduled job (`sys_trigger`) that runs a GlideRecord script server-side. This script creates:

1. `sys_ux_app_config` - links registry to config
2. `sys_ux_app_route` - maps path "home" to a screen_type
3. `sys_ux_screen_type` - defines the screen
4. `sys_ux_screen` - links screen_type to macroponent (THE MISSING LINK)

See `references/record-chain.md` for the full script template.

### Step 4: Verify

Navigate to `https://<instance>/now/<workspace-path>/home` and confirm:
- Workspace shell loads (header visible with workspace title)
- URL redirects to `/home` route
- Page content renders inside the shell viewport

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| 404 or "invalid" registry | Missing `root_macroponent` or `parent_app` on page_registry | Set root_macroponent to Breadcrumb App Shell sys_id |
| Shell loads, content blank | Missing `sys_ux_screen` record | Create screen linking screen_type to macroponent |
| Route not found | Missing `sys_ux_app_route` | Create route with correct `app_id` and `screen_type` |
| Widget not rendering | Bad composition JSON on macroponent | Verify composition structure matches working examples |

## API Patterns

### Direct REST (works for most tables)
```bash
curl -s -u "admin:PASSWORD" \
  "https://INSTANCE.service-now.com/api/now/table/TABLE_NAME" \
  -X POST -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{ ... }'
```

### Scheduled Job Workaround (for ACL-blocked tables)
```bash
curl -s -u "admin:PASSWORD" \
  "https://INSTANCE.service-now.com/api/now/table/sys_trigger" \
  -X POST -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{
    "name": "Temp - Create Workspace Records",
    "trigger_type": "0",
    "script": "// GlideRecord script here",
    "next_action": "DATETIME_30_SECONDS_FROM_NOW",
    "state": "0",
    "run_type": "once"
  }'
```

### Reverse-Engineering a Working Workspace
When stuck, query a known-good workspace (e.g., CMDB) and replicate its structure:
```bash
# Get CMDB workspace registry
curl -s -u "admin:PWD" "https://INSTANCE.service-now.com/api/now/table/sys_ux_page_registry?sysparm_query=pathLIKEcmdb-workspace&sysparm_fields=sys_id,title,root_macroponent,parent_app,admin_panel"

# Get its routes
curl -s -u "admin:PWD" "https://INSTANCE.service-now.com/api/now/table/sys_ux_app_route?sysparm_query=app_id=SYS_ID&sysparm_fields=sys_id,path,screen_type,viewport_element_id"

# Get its screen_types and screens
curl -s -u "admin:PWD" "https://INSTANCE.service-now.com/api/now/table/sys_ux_screen?sysparm_query=screen_type=SCREEN_TYPE_ID&sysparm_fields=sys_id,macroponent,screen_type"
```

## Important Notes

- Always study a working workspace first before building from scratch
- The `sys_ux_screen` table is the most commonly missed record
- Breadcrumb App Shell is the standard workspace chrome/shell macroponent
- Viewport element ID on routes should typically be empty when using Breadcrumb Shell
- For full record structures and script templates, read `references/record-chain.md`
