# AI_TOOL_GUIDE.md — Roblox AI Executor Bridge Reference

> **Purpose:** Upload this file at the start of a new AI conversation so the AI agent knows how to authenticate, register, discover tools, inspect a live Roblox environment, navigate, interact with objects, monitor progress, and handle errors safely.
>
> **Security Rule:** Never hardcode or print real tokens (`AI_API_TOKEN`, `EXECUTOR_API_TOKEN`, `DASHBOARD_ADMIN_TOKEN`) inside prompts, logs, or saved files. Always receive the `AI_API_TOKEN` and base URL securely at runtime from the user or environment.

---

## 1. System Prompt & Operating Rules for Future AI Agents

When operating through the Roblox AI Executor Bridge, you MUST obey these principles:

1. **Real Environment:** You are connected to a live Roblox game session via the HTTP bridge and `BridgeClient.lua`. Tool outputs come from actual Roblox execution — never fabricate or guess game state.
2. **Inspect Before Acting:** Always start a session by calling `GetGameInfo`, `GetLocalPlayerInfo`, `GetCharacterState`, and `GetWorkspaceSummary` (or `GetNearbyInteractables`) before attempting movement or interaction.
3. **Never Invent Names, Paths, or Coordinates:** Only pass instance paths (`Workspace.VaultDoor...`), prompt paths, GUI paths, or Vector3 coordinates that you observed in real tool responses.
4. **Use Specific Tools First:** Prefer structured tools (`GetNearbyPrompts`, `FindInstances`, `MoveTo`, `InteractWithPrompt`) over `ExecuteLuau`. Only use `ExecuteLuau` for diagnostics when structured tools cannot answer a question.
5. **Verify Important Actions:** After calling `MoveTo`, `InteractWithPrompt`, or `ClickVisibleGuiButton`, check `GetCharacterState`, `GetVisibleGuiTree`, or `GetNearbyInteractables` to confirm the action actually succeeded.
6. **Respect Rate Limits & Bounds:** Keep search radii and `maxResults` reasonable (`radius <= 150`, `maxResults <= 25` unless needed). Do not spam identical calls in tight loops.
7. **Report Errors Accurately:** If a tool returns `ok: false`, a timeout, `EXECUTOR_OFFLINE`, or `APPROVAL_REQUIRED`, report the exact error code and message to the user.
8. **Stay Within the Authorised Experience:** Respect `ALLOWED_PLACE_IDS`, never attempt to fetch external URLs via tool arguments, and stop immediately if the user triggers an emergency stop.

---

## 2. What the System Does & How It Connects

The bridge runs on Node.js (TCP port `3003`) using **HTTP request/response and polling** (no WebSockets):

1. The Roblox executor runs `executor/BridgeClient.lua`, authenticates with `EXECUTOR_API_TOKEN`, and polls for tasks.
2. You (the AI agent) authenticate with `AI_API_TOKEN`, register as the AI side (`POST /api/v1/ai/register`), and discover tools (`GET /api/v1/ai/tools`).
3. When you call `POST /api/v1/ai/tool-call`, the server validates your arguments, queues the task with a unique `requestId`, dispatches it to the paired executor, waits for the correlated result, and returns the real Roblox response.

### Connection URL Options
- **Over WireGuard VPN (Recommended when no domain/IP cert):** `http://10.66.66.1:3003`
- **Over Public HTTPS (when IP TLS certificate is configured):** `https://77.68.33.65:3003`

---

## 3. Authentication & Registration

Include your `AI_API_TOKEN` in the `Authorization` header on every request:

```http
Authorization: Bearer <YOUR_AI_API_TOKEN>
Content-Type: application/json
```

Optional replay protection headers (recommended):
```http
X-Request-Timestamp: 1763230000000
X-Request-Nonce: unique-random-string-at-least-8-chars
```

### Step 1: Check Server Info
```http
GET /api/v1/info
Authorization: Bearer <YOUR_AI_API_TOKEN>
```

### Step 2: Register as the AI Side
```http
POST /api/v1/ai/register
Authorization: Bearer <YOUR_AI_API_TOKEN>
Content-Type: application/json

{
  "clientName": "AutonomousExplorerAI"
}
```
*(Optional: pass `"sessionId": "sess_..."` if attaching to a specific existing session.)*

**Example Response:**
```json
{
  "ok": true,
  "sessionId": "sess_8c4b0b78-4f6c-4a3b-9b4a-8c9d6e123456",
  "aiClientId": "ai_91f0a3c2b1e4",
  "executorConnected": true,
  "executor": {
    "executorName": "Synapse / Wave",
    "placeId": 123456789,
    "jobId": "a1b2c3d4",
    "gameName": "Authorised Test Arena",
    "playerName": "PlayerOne",
    "capabilities": {
      "fireproximityprompt": true,
      "firetouchinterest": true,
      "fireclickdetector": true,
      "loadstring": true
    }
  },
  "policy": {
    "executeLuauEnabled": true,
    "executeLuauRequireApproval": true
  },
  "toolCount": 43
}
```

If `executorConnected` is `false`, tell the user to execute `BridgeClient.lua` inside their Roblox executor and wait before calling workspace tools.

---

## 4. Discovering Tools

```http
GET /api/v1/ai/tools
Authorization: Bearer <YOUR_AI_API_TOKEN>
```
- Add `?format=openai` to also receive OpenAI-compatible `function` definitions.
- Add `?format=mcp` or `?format=all` for MCP `inputSchema` definitions.
- An **MCP over HTTP (JSON-RPC 2.0)** endpoint is also available at `POST /mcp` (`initialize`, `tools/list`, `tools/call`).

---

## 5. Calling Tools, Async Monitoring, and Cancellation

### Synchronous Tool Call (default `wait: true`)
```http
POST /api/v1/ai/tool-call
Authorization: Bearer <YOUR_AI_API_TOKEN>
Content-Type: application/json

{
  "sessionId": "sess_8c4b0b78-4f6c-4a3b-9b4a-8c9d6e123456",
  "toolName": "GetCharacterState",
  "args": {},
  "timeoutMs": 15000
}
```

**Success Response:**
```json
{
  "ok": true,
  "requestId": "req_e2a1f456-...",
  "sessionId": "sess_8c4b0b78-...",
  "toolName": "GetCharacterState",
  "status": "completed",
  "durationMs": 14,
  "truncated": false,
  "data": {
    "alive": true,
    "position": { "x": 12.5, "y": 5.0, "z": -8.2 },
    "lookVector": { "x": 0.0, "y": 0.0, "z": -1.0 },
    "health": 100,
    "maxHealth": 100,
    "walkSpeed": 16,
    "humanoidState": "Enum.HumanoidStateType.Running"
  }
}
```

### Asynchronous Tool Call & Status Polling (`wait: false`)
Useful for long-running navigation (`MoveTo`) or `ExecuteLuau` awaiting manual approval:
```json
{
  "sessionId": "sess_...",
  "toolName": "MoveTo",
  "args": { "targetPosition": { "x": 45, "y": 5, "z": -30 } },
  "wait": false
}
```
Poll status with:
```http
GET /api/v1/ai/requests/<requestId>?sessionId=<sessionId>
Authorization: Bearer <YOUR_AI_API_TOKEN>
```

### Cancelling a Request or Stopping Movement
- Cancel a specific pending/running request:
```http
POST /api/v1/ai/requests/<requestId>/cancel
Authorization: Bearer <YOUR_AI_API_TOKEN>
Content-Type: application/json

{ "sessionId": "sess_...", "reason": "Target changed" }
```
- Stop character movement immediately: call `StopMovement` or `CancelMovement`.
- Disconnect cleanly when finished: `POST /api/v1/session/disconnect` with `{ "sessionId": "sess_...", "target": "ai" }`.

---

## 6. Complete Tool Catalog (All 43 Tools)

### A. Environment Inspection (1–15)

| # | Tool Name | Purpose | Required Args | Optional Args |
|---|---|---|---|---|
| 1 | `GetGameInfo` | Current game name, PlaceId, GameId, JobId, player count | — | — |
| 2 | `GetLocalPlayerInfo` | LocalPlayer name, displayName, UserId, Team, character path | — | — |
| 3 | `GetCharacterState` | Position, lookVector, velocity, health, walkSpeed, Humanoid state | — | — |
| 4 | `GetCameraState` | Camera position, lookVector, FOV, CameraType, viewport | — | — |
| 5 | `GetWorkspaceSummary` | Top-level children & class counts in Workspace | — | `maxTopLevelItems` (1–100) |
| 6 | `GetDescendants` | Paginated descendants with class filter & depth limit | — | `rootPath`, `classFilter`, `maxDepth` (1–12), `limit` (1–100), `offset` |
| 7 | `GetChildren` | Direct children of an instance | `instancePath` | `classFilter`, `limit` (1–100) |
| 8 | `FindInstances` | Find instances by name substring and/or ClassName | *(at least one of `nameContains`, `className`)* | `nameContains`, `className`, `rootPath`, `maxResults` |
| 9 | `SearchWorkspace` | Search Workspace names & prompt texts by keyword | `query` | `classFilter`, `maxResults` |
| 10 | `GetInstanceInfo` | Name, ClassName, path, parentPath, position, tags, prompt/part details | `instancePath` | — |
| 11 | `GetInstanceProperties` | Read allowlisted safe properties (`Name`, `Position`, `Text`, `Enabled`, etc.) | `instancePath`, `properties` (array) | — |
| 12 | `GetInstanceAttributes` | Read custom Roblox Attributes on an instance | `instancePath` | — |
| 13 | `GetCollectionServiceTags` | Get tags on an instance or instances with a tag | *(at least one of `instancePath`, `tagName`)* | `instancePath`, `tagName`, `maxResults` |
| 14 | `GetLightingInfo` | ClockTime, Brightness, Ambient, FogStart, FogEnd, GlobalShadows | — | — |
| 15 | `GetVisibleGuiTree` | Inspect visible PlayerGui hierarchy, readable text, and buttons | — | `maxDepth` (1–10), `maxNodes` (1–150), `includeInvisible` |

### B. Surrounding Environment (16–24)

| # | Tool Name | Purpose | Required Args | Optional Args |
|---|---|---|---|---|
| 16 | `GetNearbyParts` | Nearby `BasePart` objects sorted by distance | — | `radius` (1–500), `maxResults` (1–100), `origin` `{x,y,z}`, `nameContains` |
| 17 | `GetNearbyModels` | Nearby `Model` objects sorted by distance | — | `radius`, `maxResults`, `origin`, `nameContains` |
| 18 | `GetNearbyPlayers` | Nearby players' names, distance, position, health (no private data) | — | `radius` (1–500), `maxResults` (1–50) |
| 19 | `GetNearbyInteractables` | Nearby ProximityPrompts, ClickDetectors, TouchInterests, Seats | — | `radius`, `maxResults` |
| 20 | `GetNearbyPrompts` | Nearby ProximityPrompts with ActionText, ObjectText, distance | — | `radius`, `maxResults`, `onlyEnabled` |
| 21 | `GetNearestObject` | Nearest object matching name, class, or tag | *(at least one of `nameContains`, `className`, `tagName`)* | `nameContains`, `className`, `tagName`, `maxRadius` |
| 22 | `GetObjectsInRadius` | Matching BaseParts/Models/Prompts inside radius | `radius` | `origin`, `nameContains`, `className`, `maxResults` |
| 23 | `GetNavigationPath` | Compute `PathfindingService` path & waypoints to destination | *(one of `targetPosition`, `targetPath`)* | `targetPosition`, `targetPath`, `agentRadius`, `agentHeight`, `agentCanJump` |
| 24 | `GetMapOverview` | SpawnLocations and major top-level landmarks/areas | — | `maxLandmarks` (5–60) |

### C. Movement and Navigation (25–35)

| # | Tool Name | Purpose | Required Args | Optional Args |
|---|---|---|---|---|
| 25 | `MoveTo` | Walk smoothly using Humanoid:MoveTo + PathfindingService | *(one of `targetPosition`, `targetPath`)* | `targetPosition`, `targetPath`, `usePathfinding`, `stopDistance`, `timeoutSeconds`, `waitForArrival` |
| 26 | `MoveDirection` | Move in a direction vector for a limited duration | `direction` `{x,y,z}` | `durationSeconds` (0.1–10), `relativeToCamera` |
| 27 | `StopMovement` | Immediately halt active movement | — | — |
| 28 | `Jump` | Trigger normal Humanoid jump | — | — |
| 29 | `FacePosition` | Rotate character horizontally toward a position or instance | *(one of `targetPosition`, `targetPath`)* | `targetPosition`, `targetPath` |
| 30 | `FollowPath` | Walk through an explicit array of `{x,y,z}` waypoints | `waypoints` (array) | `stopDistance`, `timeoutSeconds` |
| 31 | `CancelMovement` | Cancel active navigation task | — | — |
| 32 | `GetMovementStatus` | Current movement mode, target, waypoints remaining, speed | — | — |
| 33 | `GetDistanceTo` | Distance & vertical delta to a position or instance | *(one of `targetPosition`, `targetPath`)* | `targetPosition`, `targetPath` |
| 34 | `WaitForObject` | Wait for a named object to appear under `rootPath` | `name` | `rootPath`, `timeoutSeconds` (0.5–25) |
| 35 | `WaitForStateChange` | Wait for `humanoid_state`, `health`, `position_moved`, or `attribute` change | `watchType` | `instancePath`, `attributeName`, `timeoutSeconds` (0.5–25) |

### D. Interaction (36–42)

| # | Tool Name | Purpose | Required Args | Optional Args |
|---|---|---|---|---|
| 36 | `InteractWithPrompt` | Fire a `ProximityPrompt` via `fireproximityprompt` | `promptPath` | `holdDurationOverride` |
| 37 | `InspectInteractable` | Inspect prompts/clickdetectors/touchinterests (optionally trigger) | `instancePath` | `triggerTouchInterest`, `triggerClickDetector` |
| 38 | `InspectGui` | Inspect a specific ScreenGui or GuiObject by path | `guiPath` | `maxDepth` (1–10) |
| 39 | `ClickVisibleGuiButton` | Click a visible `GuiButton` via normal input behaviour | `buttonPath` | — |
| 40 | `GetAvailableTools` | List equipped and Backpack `Tool` items (optionally equip one) | — | `equipToolName` |
| 41 | `GetCurrentTasks` | Read leaderstats and visible quest/task HUD labels | — | — |
| 42 | `GetExecutorCapabilities` | Check executor identity & supported functions | — | — |

### E. Code Execution (43)

| # | Tool Name | Purpose | Required Args | Optional Args |
|---|---|---|---|---|
| 43 | `ExecuteLuau` | Run diagnostic Luau code in authorised test environment (manual dashboard approval enabled by default) | `code` (<= 16 KB) | `timeoutMs` (250–15000), `reason` |

---

## 7. Complete Example Workflow: Inspect, Navigate, and Interact

1. **Get Game & Character State:**
   Call `GetGameInfo` and `GetCharacterState` to confirm you are alive and where you stand (`{ x: 0, y: 5, z: 0 }`).
2. **Find Nearby Interactables:**
   ```json
   { "sessionId": "sess_...", "toolName": "GetNearbyInteractables", "args": { "radius": 80 } }
   ```
   Result shows `Workspace.VaultDoor.DoorPart.DoorPrompt` at `{ x: 24, y: 5, z: -12 }`, distance `26.83` studs.
3. **Check Navigation Path:**
   ```json
   { "sessionId": "sess_...", "toolName": "GetNavigationPath", "args": { "targetPosition": { "x": 24, "y": 5, "z": -12 } } }
   ```
   Result shows `"available": true`, `"waypointCount": 3`.
4. **Navigate to the Object:**
   ```json
   { "sessionId": "sess_...", "toolName": "MoveTo", "args": { "targetPosition": { "x": 24, "y": 5, "z": -12 }, "stopDistance": 4 } }
   ```
   Result shows `"arrived": true`.
5. **Verify Position & Interact:**
   Call `GetCharacterState` to confirm you are near `{ x: 24, y: 5, z: -12 }`, then call:
   ```json
   { "sessionId": "sess_...", "toolName": "InteractWithPrompt", "args": { "promptPath": "Workspace.VaultDoor.DoorPart.DoorPrompt" } }
   ```
6. **Observe Outcome:**
   Call `GetNearbyInteractables` or `GetVisibleGuiTree` to see what changed in the environment.

---

## 8. Interpreting Errors & Security Restrictions

| Error Code | Meaning | What You Should Do |
|---|---|---|
| `UNAUTHORIZED` | Invalid/missing `AI_API_TOKEN` | Ask the user to verify the token; never print tokens. |
| `EXECUTOR_OFFLINE` | Executor disconnected or not running | Ask the user to start/reconnect `BridgeClient.lua`. |
| `PLACE_NOT_ALLOWED` | Current PlaceId is blocked by `ALLOWED_PLACE_IDS` | Inform the user the current experience is not on the allowlist. |
| `TOOL_DISABLED` | `ExecuteLuau` is disabled in the dashboard | Use structured tools instead, or ask the user to enable it. |
| `REQUEST_REJECTED` | User rejected an `ExecuteLuau` request in the dashboard | Respect the rejection and adjust your plan. |
| `REQUEST_TIMEOUT` | Executor did not respond in time | Check `GetMovementStatus` / `GetCharacterState` before retrying. |
| `REQUEST_CANCELLED` | Emergency stop or cancellation triggered | Halt actions and wait for user instructions. |
| `PAYLOAD_TOO_LARGE` | Result exceeded `MAX_RESULT_BYTES` | Re-run with a smaller `radius`, `limit`, or `maxDepth`. |
| `INVALID_INPUT` | Bad schema or blocked external URL | Fix arguments according to `GET /api/v1/ai/tools`. |
