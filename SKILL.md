---
name: n8n-workflow-authoring
description: Structured guide for creating and editing n8n workflow JSON files. Use when building, modifying, or debugging n8n workflows — covers node types and their JSON schemas (Schedule Trigger, HTTP Request, Code, IF, Switch, Telegram), credential binding patterns, error handling, retry configuration, persistent state via static data, and workflow validation. Invoke whenever you are authoring a new workflow from scratch, editing an existing workflow JSON, or debugging unexpected node behaviour.
---

# n8n Workflow Authoring Guide

This skill covers everything needed to produce valid, maintainable n8n workflow JSON — from top-level structure through node-by-node schemas, credential binding, error handling, persistent state, and validation.

---

## Workflow Top-Level Structure

Every n8n workflow JSON file has this shape:

```json
{
  "id": "kebab-case-unique-id",
  "name": "Human-Readable Workflow Name",
  "nodes": [ /* array of node objects */ ],
  "connections": { /* wiring between nodes */ },
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "staticData": null
}
```

- `id` — unique, kebab-case string. Used for import deduplication.
- `active` — set to `false` in JSON files; activate through the n8n UI after import.
- `settings.executionOrder: "v1"` — required for consistent expression evaluation order in n8n ≥ 1.0.

---

## Node Object Shape

Every node shares this base schema:

```json
{
  "id": "unique-node-id",
  "name": "Display Name",
  "type": "n8n-nodes-base.nodeType",
  "typeVersion": 1.2,
  "position": [x, y],
  "parameters": { /* node-specific config */ },
  "credentials": { /* optional, only for nodes that use auth */ },
  "onError": "continueRegularOutput",
  "retryOnFail": false
}
```

Position is a `[x, y]` pixel pair. Convention: first node at `[0, 0]`, each subsequent node 220px to the right on the same row. Branch nodes use vertical offsets (e.g., `[440, -100]` and `[440, 100]`).

---

## Standard Node Types

### Schedule Trigger

Fires the workflow on a cron-like interval. No credentials required.

```json
{
  "id": "schedule-trigger",
  "name": "Every 5 Minutes",
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.2,
  "position": [0, 0],
  "parameters": {
    "rule": {
      "interval": [
        {
          "field": "minutes",
          "minutesInterval": 5
        }
      ]
    }
  }
}
```

`field` options: `"minutes"`, `"hours"`, `"days"`, `"weeks"`, `"months"`, `"cronExpression"`.

For cron expressions:

```json
"rule": {
  "interval": [
    {
      "field": "cronExpression",
      "expression": "0 9 * * 1-5"
    }
  ]
}
```

---

### HTTP Request

Makes outbound HTTP calls. Credential binding is optional — omit `credentials` for unauthenticated requests.

```json
{
  "id": "http-call",
  "name": "Call API",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [220, 0],
  "parameters": {
    "method": "GET",
    "url": "https://example.com/api/health",
    "options": {
      "timeout": 10000,
      "fullResponse": true
    }
  },
  "onError": "continueRegularOutput",
  "retryOnFail": false
}
```

Key parameters:
- `method` — `"GET"`, `"POST"`, `"PATCH"`, `"DELETE"` (default: `"GET"`)
- `url` — full URL string or n8n expression `={{ $vars.MY_URL }}`
- `options.timeout` — milliseconds; 10000 is safe for health checks, 30000 for slow APIs
- `options.fullResponse: true` — returns `{ statusCode, headers, body }` instead of just the body; required when you need to inspect the status code in downstream Code nodes
- `onError: "continueRegularOutput"` — passes the error as a normal output item rather than halting the workflow; use this when the Code node downstream handles failure logic
- `retryOnFail` — set to `true` + `maxTries`/`waitBetweenTries` only for idempotent requests where a transient network error should be retried automatically

**With request body (POST):**

```json
"parameters": {
  "method": "POST",
  "url": "https://api.example.com/items",
  "contentType": "json",
  "body": {
    "values": {
      "string": [
        { "name": "key", "value": "={{ $json.value }}" }
      ]
    }
  },
  "options": { "timeout": 15000 }
}
```

**With query parameters:**

```json
"parameters": {
  "url": "https://api.example.com/search",
  "sendQuery": true,
  "queryParameters": {
    "parameters": [
      { "name": "q", "value": "={{ $json.searchTerm }}" },
      { "name": "limit", "value": "50" }
    ]
  }
}
```

---

### Code (JavaScript)

Runs arbitrary JavaScript. The main node for custom logic, data transformation, and evaluation.

```json
{
  "id": "code-evaluate",
  "name": "Evaluate Result",
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [440, 0],
  "parameters": {
    "jsCode": "// Your code here\nreturn [{ json: { result: 'ok' } }];"
  }
}
```

**Must return** an array of items: `[{ json: { ... } }]`.

**Accessing upstream node output:**

```javascript
// From a specific node by name
const item = $('Node Name').first().json;
const statusCode = item?.statusCode || item?.$response?.statusCode;

// From the direct input
const input = $input.first().json;

// All items from a node
const items = $('Node Name').all();
```

**Persistent state with static data** (survives across executions):

```javascript
const staticData = $getWorkflowStaticData('global');

// Initialize defaults on first run
if (staticData.consecutiveFailures === undefined) staticData.consecutiveFailures = 0;
if (staticData.lastAlertSentAt === undefined) staticData.lastAlertSentAt = 0;

// Mutations are persisted automatically — no save call needed
staticData.consecutiveFailures++;
```

`$getWorkflowStaticData('global')` returns a mutable object. Any property you set is saved to the workflow's static data store at the end of the execution. Use this for: failure counters, last-alert timestamps, state flags (e.g., `inFailureState`).

**Building multi-line Telegram message strings** (avoid template literals with `\n` injection issues):

```javascript
const message = [
  '🚨 *Alert Title*',
  '',
  `*Field:* ${value}`,
  `*Time:* ${new Date().toISOString()}`,
].join('\n');
```

---

### IF (Binary Branch)

Routes items down `true` or `false` output based on a condition.

```json
{
  "id": "if-check",
  "name": "Is Healthy?",
  "type": "n8n-nodes-base.if",
  "typeVersion": 2,
  "position": [440, 0],
  "parameters": {
    "conditions": {
      "options": {
        "caseSensitive": true,
        "leftValue": "",
        "typeValidation": "strict"
      },
      "conditions": [
        {
          "id": "cond-1",
          "leftValue": "={{ $json.status }}",
          "rightValue": "ok",
          "operator": {
            "type": "string",
            "operation": "equals"
          }
        }
      ],
      "combinator": "and"
    }
  }
}
```

Operator options: `equals`, `notEquals`, `contains`, `startsWith`, `endsWith`, `regex`, `larger`, `smaller`, `largerEqual`, `smallerEqual`.

Output 0 = `true` branch, Output 1 = `false` branch.

---

### Switch (Multi-Branch)

Routes items to named outputs based on multiple conditions. Use instead of nested IFs.

```json
{
  "id": "switch-action",
  "name": "Route by Action",
  "type": "n8n-nodes-base.switch",
  "typeVersion": 3,
  "position": [660, 0],
  "parameters": {
    "mode": "rules",
    "rules": {
      "values": [
        {
          "outputKey": "alert",
          "conditions": {
            "options": { "caseSensitive": true, "leftValue": "", "typeValidation": "strict" },
            "conditions": [
              {
                "id": "cond-alert",
                "leftValue": "={{ $json.action }}",
                "rightValue": "alert",
                "operator": { "type": "string", "operation": "equals" }
              }
            ],
            "combinator": "and"
          }
        },
        {
          "outputKey": "recovery",
          "conditions": {
            "options": { "caseSensitive": true, "leftValue": "", "typeValidation": "strict" },
            "conditions": [
              {
                "id": "cond-recovery",
                "leftValue": "={{ $json.action }}",
                "rightValue": "recovery",
                "operator": { "type": "string", "operation": "equals" }
              }
            ],
            "combinator": "and"
          }
        }
      ]
    },
    "options": {
      "fallbackOutput": "none"
    }
  }
}
```

`outputKey` values become the named outputs. `fallbackOutput: "none"` drops items that match no rule (use `"extra"` to route unmatched items to an extra output).

---

### Telegram (Send Message)

Sends a message to a Telegram chat. Requires a Telegram Bot credential.

```json
{
  "id": "telegram-send",
  "name": "Send Alert",
  "type": "n8n-nodes-base.telegram",
  "typeVersion": 1.2,
  "position": [880, 0],
  "parameters": {
    "operation": "sendMessage",
    "chatId": "={{ $vars.TELEGRAM_CHAT_ID }}",
    "text": "={{ $json.message }}",
    "additionalFields": {
      "parse_mode": "Markdown"
    }
  },
  "credentials": {
    "telegramApi": {
      "id": "CREDENTIAL_ID",
      "name": "Telegram Bot"
    }
  }
}
```

- `chatId` — use `$vars.TELEGRAM_CHAT_ID` (set in n8n UI under Variables) rather than hardcoding the ID
- `parse_mode: "Markdown"` — enables `*bold*`, `_italic_`, backtick code spans
- The `text` field receives the message string; build it in a Code node upstream and pass via `$json.message`

---

## Credential Binding Patterns

**Never embed credentials (tokens, passwords, bot tokens) in workflow JSON.** n8n has a first-class credential store.

### Reference by credential name

When you know the credential exists in n8n, reference it by ID and name:

```json
"credentials": {
  "telegramApi": {
    "id": "1",
    "name": "Telegram Bot"
  },
  "httpBasicAuth": {
    "id": "2",
    "name": "Paperclip API Key"
  }
}
```

The `id` is the numeric ID assigned by n8n when the credential is created. Use a placeholder like `"CREDENTIAL_ID"` in committed workflow JSON and replace it after importing into n8n. The `name` is the human-readable label from the n8n Credentials UI.

### Using n8n Variables for dynamic values

For values that change per environment (base URLs, chat IDs, thresholds):

1. In n8n UI: Settings → Variables → add key/value pairs
2. In workflow JSON: reference as `={{ $vars.MY_VARIABLE }}`

```json
"url": "={{ $vars.PAPERCLIP_API_BASE_URL }}/api/health"
```

Variables are resolved at execution time and never appear in the workflow JSON export.

---

## Error Handling and Retry Configuration

### Continue on error

Use `onError: "continueRegularOutput"` on HTTP Request nodes when downstream Code nodes contain the failure-handling logic. This passes the error response as a normal item rather than stopping the workflow.

```json
"onError": "continueRegularOutput",
"retryOnFail": false
```

### Retry on transient failures

For idempotent requests where transient network errors should be auto-retried:

```json
"onError": "continueRegularOutput",
"retryOnFail": true,
"maxTries": 3,
"waitBetweenTries": 2000
```

`waitBetweenTries` is in milliseconds. Do not enable `retryOnFail` for non-idempotent POST/PATCH operations unless the endpoint is explicitly idempotent.

### Defensive status code parsing in Code nodes

When `fullResponse: true` is set on the HTTP Request, the Code node must defensively extract the status code — the response shape can vary:

```javascript
const healthItem = $('Health Check').first();
const statusCode = healthItem?.json?.statusCode 
  || healthItem?.json?.$response?.statusCode;
const body = healthItem?.json?.body || healthItem?.json || {};
```

---

## Connections Schema

The `connections` object maps each node's output to downstream inputs:

```json
"connections": {
  "Every 5 Minutes": {
    "main": [
      [{ "node": "Health Check", "type": "main", "index": 0 }]
    ]
  },
  "Health Check": {
    "main": [
      [{ "node": "Evaluate Health", "type": "main", "index": 0 }]
    ]
  },
  "Route by Action": {
    "main": [
      [{ "node": "Send Alert", "type": "main", "index": 0 }],
      [{ "node": "Send Recovery", "type": "main", "index": 0 }]
    ]
  }
}
```

Each `"main"` array entry corresponds to an output port. Single-output nodes have one array inside `main`. Multi-output nodes (IF, Switch) have one array per output, in port order.

For Switch nodes with named outputs, use the port index order matching the `rules.values` array order.

---

## Persistent State Patterns

### Failure counter with cooldown

```javascript
const staticData = $getWorkflowStaticData('global');

if (staticData.consecutiveFailures === undefined) staticData.consecutiveFailures = 0;
if (staticData.inFailureState === undefined) staticData.inFailureState = false;
if (staticData.lastAlertSentAt === undefined) staticData.lastAlertSentAt = 0;

const COOLDOWN_MS = 2 * 60 * 60 * 1000; // 2 hours
const now = Date.now();

if (isHealthy) {
  staticData.consecutiveFailures = 0;
  staticData.inFailureState = false;
} else {
  staticData.consecutiveFailures++;

  if (staticData.consecutiveFailures >= 3 && now - staticData.lastAlertSentAt > COOLDOWN_MS) {
    staticData.inFailureState = true;
    staticData.lastAlertSentAt = now;
    // trigger alert downstream
  }
}
```

### Reset on recovery

Always reset failure state when the service recovers so subsequent failures start a fresh counter:

```javascript
if (isHealthy && staticData.inFailureState) {
  const priorCount = staticData.consecutiveFailures;
  staticData.consecutiveFailures = 0;
  staticData.inFailureState = false;
  // send recovery notification mentioning priorCount
}
```

---

## Testing and Validation Approaches

### Before importing into n8n

1. **JSON validity**: `python3 -m json.tool workflow.json > /dev/null` — catches syntax errors.
2. **Required field check**: confirm every node has `id`, `name`, `type`, `typeVersion`, `position`, and `parameters`.
3. **Connection node name match**: every key in `connections` must exactly match a `name` field in `nodes`. Mismatches silently drop connections on import.
4. **Credential placeholder audit**: search for any hardcoded tokens, passwords, or bot tokens — they must not appear in the JSON file.

### After importing into n8n

1. Open the workflow in the n8n UI and confirm all nodes render without error icons.
2. Re-bind credentials: click each node that shows a missing credential and select the correct stored credential.
3. Set any `$vars.*` values used in expressions under Settings → Variables.
4. Use "Execute Workflow" (manual trigger) to run a single test execution and inspect each node's output.
5. Check the Code node outputs match expected schema before activating.

### Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| `connections` key doesn't match node `name` | Node appears disconnected after import | Sync the key to the exact `name` string |
| `onError` not set on HTTP node | Workflow halts on first HTTP error | Add `"onError": "continueRegularOutput"` |
| `fullResponse: true` omitted | Code node can't read `statusCode` | Add to `options` on the HTTP Request |
| Static data mutation on wrong scope | State not persisted | Use `$getWorkflowStaticData('global')`, not `'node'` |
| Hardcoded chat ID | Alert stops working after environment change | Move to `$vars.TELEGRAM_CHAT_ID` |
| Message built with template literals and `\n` | Markdown newlines not rendered | Build as array and `.join('\n')` |
| `typeVersion` wrong | Node parameters ignored or defaults applied | Check n8n docs for current version (httpRequest: 4.2, code: 2, switch: 3) |

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
