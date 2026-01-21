# Auto-Enabled Plugins

## Overview

Plugins can be configured to automatically be available to the AI without requiring users to manually select them in each chat session. This allows the AI to intelligently decide when to use tools like search based on the user's question.

## How It Works

```
User asks a question
         ↓
AI receives all auto-enabled plugins as available tools
         ↓
AI decides whether to use the tool based on the question
         ↓
If needed, AI calls the tool (e.g., SearXNG search)
```

## Configuration

### In plugins.json

Add `"autoEnabled": true` to any plugin that should always be available:

```json
{
  "id": "searxng",
  "name": "SearXNGSearch",
  "schema": "/searxng-openapi.json",
  "autoEnabled": true
}
```

### Plugin Type Definition

The `autoEnabled` field is defined in `app/store/plugin.ts`:

```typescript
export type Plugin = {
  id: string;
  createdAt: number;
  title: string;
  version: string;
  content: string;
  builtin: boolean;
  authType?: string;
  authLocation?: string;
  authHeader?: string;
  authToken?: string;
  autoEnabled?: boolean; // If true, this plugin is always available to AI
};
```

## Implementation Details

The `getAsTools` method in `app/store/plugin.ts` handles auto-enabled plugins:

1. Gets manually selected plugins from the current session
2. Gets all plugins with `autoEnabled: true` (excluding duplicates)
3. Combines both lists and returns them as available tools

```typescript
getAsTools(ids: string[]) {
  const plugins = get().plugins;
  // Get manually selected plugins
  const manuallySelected = (ids || [])
    .map((id) => plugins[id])
    .filter((i) => i);
  // Get auto-enabled plugins that aren't already selected
  const autoEnabled = Object.values(plugins).filter(
    (p) => p.autoEnabled && !ids?.includes(p.id),
  );
  // Combine and add to service
  const selected = [...manuallySelected, ...autoEnabled].map((p) =>
    FunctionToolService.add(p),
  );
  return [
    selected.reduce((s, i) => s.concat(i.tools), []),
    selected.reduce((s, i) => Object.assign(s, i.funcs), {}),
  ];
}
```

## Currently Auto-Enabled Plugins

| Plugin | Description |
|--------|-------------|
| SearXNG | Web search via self-hosted SearXNG instance |

## Use Cases

- **Search**: AI can search the web when users ask about current events, news, or real-time information
- **Image Generation**: Could auto-enable DALL-E for requests involving image creation
- **Code Execution**: Could auto-enable code interpreters for programming questions

## Notes

- Auto-enabled plugins still require proper authentication configuration
- Users can still manually select/deselect plugins per session if needed
- The AI model decides whether to actually use the tool based on context
