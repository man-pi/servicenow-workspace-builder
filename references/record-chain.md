# ServiceNow Workspace Record Chain - Full Reference

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Record Chain Detail](#record-chain-detail)
3. [Macroponent Composition JSON](#macroponent-composition-json)
4. [Scheduled Job Script Template](#scheduled-job-script-template)
5. [REST API Examples](#rest-api-examples)
6. [Reverse-Engineering Checklist](#reverse-engineering-checklist)

---

## Architecture Overview

A Next Experience workspace requires these linked records (in dependency order):

```
sys_ux_page_registry          <- The workspace app entry (has path, title)
  |-- root_macroponent        <- Points to "Breadcrumb App Shell" (the chrome)
  |-- parent_app              <- Parent application reference
  |-- admin_panel             <- Admin panel reference
  |
  +-- sys_ux_app_config       <- Configuration record linking to registry
  |
  +-- sys_ux_app_route        <- URL route (e.g. path="home")
        |-- screen_type       <- Points to sys_ux_screen_type
        |-- viewport_element_id <- Usually EMPTY for Breadcrumb Shell
        |
        +-- sys_ux_screen_type  <- Defines the screen
              |
              +-- sys_ux_screen <- THE GLUE: links screen_type to macroponent
                    |-- macroponent <- Points to your custom widget
                    |-- screen_type <- Points back to the screen_type
```

### Why This Is Hard

UI Builder normally auto-generates all intermediate records. When building via API, you must create each one manually and link them correctly. Missing the `sys_ux_screen` record is the #1 cause of "shell loads but content is blank."

---

## Record Chain Detail

### 1. sys_ux_macroponent (The Widget)

Key fields:
- `name`: Display name
- `composition`: JSON array defining child components
- `root_component`: Reference to base component (e.g., Stylized Text)
- `category`: "custom" for custom widgets
- `props`: JSON defining configurable properties
- `data_resources`: JSON for data bindings
- `dispatched_events`, `handled_events`, `internal_event_mappings`: Event wiring

### 2. sys_ux_page_registry (The Workspace App)

Key fields:
- `title`: Workspace display name
- `path`: URL path slug (e.g., "fed-workspace" for /now/fed-workspace)
- `root_macroponent`: sys_id of "Breadcrumb App Shell" macroponent
- `parent_app`: sys_id of parent application
- `admin_panel`: sys_id of admin panel
- `description`: Workspace description

To find Breadcrumb App Shell sys_id:
```bash
curl -s -u "admin:PWD" "INSTANCE/api/now/table/sys_ux_macroponent?sysparm_query=nameLIKEBreadcrumb%20App%20Shell&sysparm_fields=sys_id,name&sysparm_limit=1"
```

### 3. sys_ux_app_config

Key fields:
- `app_registry`: sys_id of the page_registry record
- `name`: Config name (usually same as workspace name)

### 4. sys_ux_app_route

Key fields:
- `app_id`: sys_id of the page_registry
- `path`: Route path (e.g., "home")
- `screen_type`: sys_id of the screen_type
- `viewport_element_id`: Usually empty for Breadcrumb Shell routes
- `order`: Display order (e.g., 100)
- `fields`: Additional config

### 5. sys_ux_screen_type

Key fields:
- `name`: Screen type name
- `description`: What this screen does

### 6. sys_ux_screen (THE CRITICAL MISSING LINK)

Key fields:
- `screen_type`: sys_id of the screen_type record
- `macroponent`: sys_id of your custom macroponent/widget
- `name`: Screen name

This record is what actually tells the workspace "when this route/screen_type is active, render THIS macroponent." Without it, the workspace shell loads but the viewport stays empty.

---

## Macroponent Composition JSON

### Simple Stylized Text Widget

```json
[
  {
    "definition": {
      "id": "STYLIZED_TEXT_COMPONENT_SYS_ID",
      "type": "COMPONENT"
    },
    "elementId": "hello_text_1",
    "elementLabel": "Hello Text",
    "propertyValues": {
      "html": {
        "type": "TRANSFORM",
        "value": {
          "script": "return '<div style=\"padding:20px;text-align:center;\"><h1>Hello from Custom Widget!</h1><p>This widget was created programmatically.</p></div>';"
        }
      }
    },
    "styles": {},
    "slot": null,
    "eventMappings": [],
    "isHidden": null,
    "preset": null
  }
]
```

To find Stylized Text component sys_id:
```bash
curl -s -u "admin:PWD" "INSTANCE/api/now/table/sys_ux_lib_component?sysparm_query=nameLIKEStylized%20Text&sysparm_fields=sys_id,name&sysparm_limit=1"
```

---

## Scheduled Job Script Template

This GlideRecord script creates the full wiring chain. Use it inside a `sys_trigger` (scheduled job) record to bypass ACL restrictions.

Replace placeholders: REGISTRY_SYS_ID, MACROPONENT_SYS_ID

```javascript
(function() {
    var registryId = 'REGISTRY_SYS_ID';
    var macroponentId = 'MACROPONENT_SYS_ID';

    // 1. Create app_config
    var config = new GlideRecord('sys_ux_app_config');
    config.initialize();
    config.setValue('name', 'My Custom Workspace Config');
    config.setValue('app_registry', registryId);
    var configId = config.insert();
    gs.info('Created app_config: ' + configId);

    // 2. Create screen_type
    var screenType = new GlideRecord('sys_ux_screen_type');
    screenType.initialize();
    screenType.setValue('name', 'My Home Screen Type');
    screenType.setValue('description', 'Home screen for custom workspace');
    var screenTypeId = screenType.insert();
    gs.info('Created screen_type: ' + screenTypeId);

    // 3. Create route
    var route = new GlideRecord('sys_ux_app_route');
    route.initialize();
    route.setValue('app_id', registryId);
    route.setValue('path', 'home');
    route.setValue('screen_type', screenTypeId);
    route.setValue('order', 100);
    // viewport_element_id should be EMPTY for Breadcrumb Shell
    var routeId = route.insert();
    gs.info('Created route: ' + routeId);

    // 4. Create screen (THE CRITICAL GLUE)
    var screen = new GlideRecord('sys_ux_screen');
    screen.initialize();
    screen.setValue('screen_type', screenTypeId);
    screen.setValue('macroponent', macroponentId);
    screen.setValue('name', 'My Home Screen');
    var screenId = screen.insert();
    gs.info('Created screen: ' + screenId);

    gs.info('Workspace wiring complete! All 4 records created.');
})();
```

---

## REST API Examples

### Create Macroponent
```bash
curl -s -u "admin:PWD" "INSTANCE/api/now/table/sys_ux_macroponent" \
  -X POST -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{
    "name": "My Custom Widget",
    "category": "custom",
    "composition": "[JSON_COMPOSITION_ARRAY]",
    "root_component": "STYLIZED_TEXT_SYS_ID",
    "props": "{}",
    "dispatched_events": "[]",
    "handled_events": "[]",
    "internal_event_mappings": "[]",
    "data_resources": "[]"
  }'
```

### Create Page Registry
```bash
curl -s -u "admin:PWD" "INSTANCE/api/now/table/sys_ux_page_registry" \
  -X POST -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{
    "title": "My Custom Workspace",
    "path": "my-workspace",
    "description": "Custom workspace",
    "root_macroponent": "BREADCRUMB_APP_SHELL_SYS_ID",
    "parent_app": "PARENT_APP_SYS_ID",
    "admin_panel": "ADMIN_PANEL_SYS_ID"
  }'
```

### Create Scheduled Job (for ACL-blocked tables)
```bash
curl -s -u "admin:PWD" "INSTANCE/api/now/table/sys_trigger" \
  -X POST -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{
    "name": "Temp - Wire Workspace (one-time)",
    "trigger_type": "0",
    "script": "ESCAPED_GLIDERECORD_SCRIPT",
    "next_action": "DATETIME_30_SECS_FROM_NOW",
    "state": "0",
    "run_type": "once"
  }'
```

---

## Reverse-Engineering Checklist

When building a new workspace, always start by querying a working one (CMDB Workspace is a good reference):

1. **Get registry**: Query `sys_ux_page_registry` with `pathLIKEcmdb` to get sys_id, root_macroponent, parent_app, admin_panel
2. **Get app_config**: Query `sys_ux_app_config` filtered by `app_registry=REGISTRY_ID`
3. **Get routes**: Query `sys_ux_app_route` filtered by `app_id=REGISTRY_ID` to see route paths and screen_types
4. **Get screen_types**: Query `sys_ux_screen_type` using the sys_ids from routes
5. **Get screens**: Query `sys_ux_screen` filtered by `screen_type=SCREEN_TYPE_ID` to see which macroponents are linked
6. **Get macroponent composition**: Query the macroponent to see its composition JSON structure

Copy the structure exactly, replacing sys_ids with your new records.

---

## Common Field Values to Look Up

These vary per instance. Always query for them rather than hardcoding:

| What | Table | Query |
|------|-------|-------|
| Breadcrumb App Shell | sys_ux_macroponent | nameLIKEBreadcrumb App Shell |
| Stylized Text component | sys_ux_lib_component | nameLIKEStylized Text |
| Parent app (global) | sys_scope | name=Global |
| CMDB Workspace (reference) | sys_ux_page_registry | pathLIKEcmdb-workspace |
