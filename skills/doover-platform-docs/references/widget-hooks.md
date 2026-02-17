# Widget Hooks & Data Access

Platform hooks and data operations available to Doover widgets.

## Imports

All hooks are imported from `customer_site/hooks`:

```javascript
import {
  useAgent,
  useAgentChannel,
  useChannelUpdateAggregate,
  useChannelSendMessage,
  useAgentState,
  useAgentCmds,
  useAgentSendUiCmd,
  dataProvider,
} from 'customer_site/hooks';
import { useRemoteParams } from 'customer_site/useRemoteParams';
```

Additional React imports used in widgets:

```javascript
import React, { useState, useEffect, useCallback } from 'react';
import { useParams } from 'react-router';
```

## Core Hooks

### useRemoteParams()

Get URL parameters (including `agentId`) in the hooks wrapper (Layer 2):

```javascript
const { agentId } = useRemoteParams();
```

Re-export of React Router's `useParams()`. Use this in Layer 2.

### useParams()

Get URL parameters in the inner component (Layer 3):

```javascript
import { useParams } from 'react-router';
const { agentId } = useParams();
```

Use this in Layer 3 when you need agentId inside the inner component.

### useAgent(agentId)

Fetch agent data:

```javascript
const { agent } = useAgent(agentId);
```

Returns `{ agent, agents, groups, organisation_users, superusers, ...query }`.

- `agent` is `undefined` while loading — use this for loading guards in Layer 2
- `agent.name` — display name of the agent/device
- Internally fetches all agents and filters by ID

## Channel Hooks

### useAgentChannel(agentId, channelName, autoFetch?)

Subscribe to a channel's aggregate data with real-time WebSocket updates:

```javascript
const { aggregate, isLoading, attachments } = useAgentChannel(agentId, "sensor_data");
```

**Parameters:**
- `agentId` — agent ID string
- `channelName` — channel name string (e.g., `"tag_values"`, `"sensor_data"`)
- `autoFetch` — optional boolean (default `true`)

**Returns:**
- `aggregate` — the channel's current aggregate data (object), `undefined` while loading
- `isLoading` — boolean, true while fetching
- `attachments` — array of file attachments in the aggregate

**Behavior:**
- Automatically subscribes to WebSocket updates on mount
- Unsubscribes on unmount
- Aggregate updates arrive in real-time via WebSocket, no polling needed
- Query cache uses `refetchInterval: Infinity` — only WebSocket triggers updates

**Example — reading channel data:**

```javascript
function MyWidgetInner() {
  const { agentId } = useParams();
  const { aggregate, isLoading } = useAgentChannel(agentId, "tag_values");

  if (isLoading) return <div>Loading...</div>;

  return (
    <div className="p-4">
      <p className="text-sm">Temperature: {aggregate?.temperature ?? 'N/A'}</p>
      <p className="text-sm">Humidity: {aggregate?.humidity ?? 'N/A'}</p>
    </div>
  );
}
```

### useAgentState(agentId)

Fetch the `ui_state` channel aggregate:

```javascript
const { state } = useAgentState(agentId);
```

Shortcut for `useAgentChannel(agentId, "ui_state")`. Returns the UI state tree for the agent.

### useAgentCmds(agentId)

Fetch the `ui_cmds` channel aggregate:

```javascript
const { cmds } = useAgentCmds(agentId);
```

Shortcut for `useAgentChannel(agentId, "ui_cmds")`. Returns current UI commands/actions.

## Mutation Hooks

### useChannelUpdateAggregate(agentId, channelName, record_log?)

PATCH (merge) data into a channel's aggregate:

```javascript
const { mutate, isPending } = useChannelUpdateAggregate(agentId, "tag_values");

const handleSet = () => {
  mutate({ temperature: 25.5, humidity: 60 });
};
```

**Parameters:**
- `agentId` — agent ID
- `channelName` — channel name
- `record_log` — optional boolean (default `false`), set `true` to log the update server-side

**Returns:** React Query mutation object with `mutate`, `mutateAsync`, `isPending`, `isError`, `error`.

**Behavior:**
- Merges payload into existing aggregate (PATCH, not replace)
- `isPending` is `true` while the request is in flight
- WebSocket will broadcast the update to all subscribers

**Example — writing to a channel:**

```javascript
function ControlPanel() {
  const { agentId } = useParams();
  const { mutate, isPending } = useChannelUpdateAggregate(agentId, "settings");

  return (
    <button
      className="px-4 py-2 rounded-md text-sm font-medium disabled:opacity-50"
      style={{ backgroundColor: '#2563eb', color: '#fff' }}
      onClick={() => mutate({ mode: 'auto', setpoint: 22 })}
      disabled={isPending}
    >
      {isPending ? 'Saving...' : 'Apply Settings'}
    </button>
  );
}
```

### useChannelSendMessage(agentId, channelName)

Send a message (append to channel history, does not modify aggregate):

```javascript
const { mutate } = useChannelSendMessage(agentId, "commands");

mutate({ action: "restart", timestamp: Date.now() });
```

Use `sendMessage` when you want to record an event in channel history. Use `updateAggregate` when you want to change the channel's current state.

### useAgentSendUiCmd(agentId)

Send a UI command (convenience wrapper):

```javascript
const { mutate } = useAgentSendUiCmd(agentId);
mutate({ command: "refresh" });
```

Equivalent to `useChannelUpdateAggregate(agentId, "ui_cmds", true)` with logging enabled.

## Direct dataProvider Access

For operations that don't fit hooks, use the `dataProvider` singleton directly:

```javascript
import { dataProvider } from 'customer_site/hooks';

// Read channel state
const ch = await dataProvider.getChannel({ agentId, channelName: 'sensor_data' });

// Send a message (append to history)
await dataProvider.sendMessage({ agentId, channelName: 'events' }, { type: 'alert', text: 'High temp' });

// Update aggregate (PATCH merge)
await dataProvider.updateAggregate({ agentId, channelName: 'settings' }, { mode: 'manual' });
```

### dataProvider Methods

| Method | Purpose |
|--------|---------|
| `getChannel({ agentId, channelName })` | Fetch channel with aggregate data |
| `getAggregate({ agentId, channelName })` | Fetch just the aggregate |
| `getMessages({ agentId, channelName }, beforeId?)` | Fetch paginated message history |
| `sendMessage({ agentId, channelName }, data)` | POST a new message to channel history |
| `updateAggregate({ agentId, channelName }, data, record_log?)` | PATCH aggregate data |
| `getAgentConnections({ agentId })` | Fetch active connections |
| `getDataSeries({ agentId, channelName }, fieldNames, timespan)` | Fetch time-series data |

## Message History

### useChannelMessages({ agentId, channelName })

Fetch paginated message history with infinite scrolling:

```javascript
import { useChannelMessages } from 'customer_site/hooks';

const { data, hasNextPage, fetchNextPage, isLoading } = useChannelMessages({
  agentId,
  channelName: 'events',
});

// data.pages is an array of page arrays, each containing MessageStructure objects
const allMessages = data?.pages?.flat() ?? [];
```

Messages are ordered newest-first. Call `fetchNextPage()` to load older messages.

### getDataSeries(identifier, fieldNames, timespan)

Fetch time-series data (not a hook — returns a Promise):

```javascript
import { getDataSeries } from 'customer_site/hooks';

const series = await getDataSeries(
  { agentId, channelName: 'sensor_data' },
  ['temperature', 'humidity'],
  { limit: 100 }  // last 100 data points
);
```

## Permission Hooks

### useHasAgentPermission(bit)

Check if the current user has a specific permission:

```javascript
import { useHasAgentPermission, AGENT_PERMISSION } from 'customer_site/hooks';

const canControl = useHasAgentPermission(AGENT_PERMISSION.DEVICE_CONTROL);
const canViewHistory = useHasAgentPermission(AGENT_PERMISSION.HISTORY_VIEW);
```

Permission bits:
- `AGENT_PERMISSION.HISTORY_VIEW` (8) — can view message history
- `AGENT_PERMISSION.DEVICE_CONTROL` (29) — can write to aggregates, send commands

Superusers always return `true`.

### useAgentPermission()

Get the raw permission bit string for the current agent:

```javascript
const permission = useAgentPermission();
// Returns "8", "0", or undefined (superuser)
```

## Common Patterns

### Read + Write Pattern

Read channel data and provide controls to update it:

```javascript
function TagEditor() {
  const { agentId } = useParams();
  const { aggregate, isLoading } = useAgentChannel(agentId, "tag_values");
  const { mutate, isPending } = useChannelUpdateAggregate(agentId, "tag_values");
  const [tagName, setTagName] = useState('');
  const [tagValue, setTagValue] = useState('');

  const handleSet = () => {
    mutate({ [tagName]: tagValue });
  };

  return (
    <div className="flex flex-col gap-4 p-4">
      {/* Display current values */}
      {isLoading ? (
        <p className="text-sm text-slate-400">Loading...</p>
      ) : (
        <pre className="text-xs bg-slate-100 rounded-md p-2 overflow-auto">
          {JSON.stringify(aggregate, null, 2)}
        </pre>
      )}

      {/* Input controls */}
      <div className="flex items-center gap-2">
        <input
          className="flex-1 h-9 rounded-md border border-slate-200 bg-transparent px-3 text-sm"
          placeholder="Tag name"
          value={tagName}
          onChange={e => setTagName(e.target.value)}
        />
        <input
          className="flex-1 h-9 rounded-md border border-slate-200 bg-transparent px-3 text-sm"
          placeholder="Value"
          value={tagValue}
          onChange={e => setTagValue(e.target.value)}
          onKeyDown={e => e.key === 'Enter' && handleSet()}
        />
        <button
          className="h-9 px-4 rounded-md text-sm font-medium disabled:opacity-50"
          style={{ backgroundColor: '#2563eb', color: '#fff' }}
          onClick={handleSet}
          disabled={isPending}
        >
          {isPending ? 'Setting...' : 'Set'}
        </button>
      </div>
    </div>
  );
}
```

### Multi-Channel Pattern

Subscribe to multiple channels:

```javascript
function DashboardWidget() {
  const { agentId } = useParams();
  const { aggregate: sensorData } = useAgentChannel(agentId, "sensor_data");
  const { aggregate: settings } = useAgentChannel(agentId, "settings");
  const { mutate: updateSettings } = useChannelUpdateAggregate(agentId, "settings");

  return (
    <div className="flex flex-col gap-4 p-4">
      <div>Temperature: {sensorData?.temperature ?? 'N/A'}</div>
      <div>Mode: {settings?.mode ?? 'unknown'}</div>
      <button onClick={() => updateSettings({ mode: 'auto' })}>
        Set Auto Mode
      </button>
    </div>
  );
}
```

### Loading + Error States

```javascript
function MyWidgetInner() {
  const { agentId } = useParams();
  const { aggregate, isLoading, isError, error } = useAgentChannel(agentId, "data");

  if (isLoading) {
    return (
      <div className="flex items-center justify-center min-h-[200px] text-slate-400 text-sm">
        Loading...
      </div>
    );
  }

  if (isError) {
    return (
      <div className="p-4 text-sm text-red-600">
        Error loading data: {error?.message ?? 'Unknown error'}
      </div>
    );
  }

  return <div>{/* render with aggregate */}</div>;
}
```

## WebSocket Behavior

Channel hooks (`useAgentChannel`, `useAgentState`, `useAgentCmds`) automatically:
1. Subscribe to WebSocket updates on mount
2. Update React Query cache when messages or aggregate updates arrive
3. Unsubscribe on unmount (cleanup)

No manual polling or refetch logic is needed — data stays live automatically.
