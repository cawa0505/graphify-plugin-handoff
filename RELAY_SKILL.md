# Code Relay — Slim Skill (SKILL.md)

## 1. Summary

This is the **only client-facing surface** for the Code Relay implementation.
Legacy slash commands (`/init`, `/save`, etc.) and the `experimental.session.compacting` hook are **removed** — all state and tool interaction is via the relay tools registered by GraphifyMCP (backed by the embedded `graphify-plugin-handoff` crate).

## 2. Quick start

The skill assumes `graphify-plugin-handoff` is embedded in Graphify and the `relay*` tools are already registered by GraphifyMCP at startup. No manual MCP server registration is needed.

Once active, the skill provides these commands (prefixed with `!` because the session tool prefix is configurable):

| Command | Description |
|---------|------------|
| `!relayInit` | Initialize a new relay state at the current project root. |
| `!relaySave` | Register/save the current repo (defaults to `basename(cwd)`). |
| `!relayClose` | Close and commit the current repo state (performs consistency & spec sync). |
| `!relaySwitch <repo>` | Pass the baton to another registered repo. |
| `!relayStatus` | Show the current relay root, project context, active baton, and repo status. |
| `!relayResume` | Render the resume for the active baton (or `<repo>`). |
| `!relayAdd <file>` | Add a handoff file (captures its content into threads). |

## 3. Operation details (action → tool result)

The skill translates each command into a single MCP `tools/call` on the relay tool exposed by GraphifyMCP.

### 3.1 `!relayInit`

**Tool call**: `relayInit` (arguments: `project_context?, kind?`)

**Return**: text output from the server:

```
Initialized relay at <cwd>/relay.json
- specs/ and .code-relay/ created
- .gitignore updated (relay.json, RESUME.md, next_step.md)
- run relaySave to register the current repo
```

**Notes**:
- If `relay.json` already exists in any parent directory, the command fails with the same message as the legacy plugin.
- `.gitignore` updates apply **only** when cwd is a git repo; the skill still shows the line even when skipped.

### 3.2 `!relaySave`

**Tool call**: `relaySave` (args: `repo?, role?, active_phase?, volatile_state?, confidence?, next_session_starter?, debt_tag?, kind?`)

**Return**:

```
Saved state for "<repo>".
Active baton: <active_baton>

<rendered resume>
```

**Notes**:
- `debt_tag` accepts a comma-separated string; each item becomes a line in the resume.
- `confidence` is clamped to 1–5 and rounded.

### 3.3 `!relayClose`

**Tool call**: `relayClose` (args: `repo?, next_session_starter?`)

**Return** (concatenated report lines):

```
Closing ritual for "<repo>".
Consistency: OK|ISSUES
  - <issue>            (per issue, only when ISSUES)
Spec sync: <a:added,b:modified> | no changes

<rendered next_step.md>

committed: <hash> | nothing to commit (files ignored or missing) | not a git repo — skipped commit
```

**Notes**:
- A spec that includes `## REMOVED Requirements` and has non-whitespace content after triggers a consistency issue.
- If the repo is not a git repo, the `committed:` line reports accordingly.

### 3.4 `!relaySwitch <repo>`

**Tool call**: `relaySwitch` (args: `repo`, `kind?`)

**Return**:

```
Baton passed to "<repo>".

<rendered resume>
```

**Failure cases** (same text as legacy plugin):
- `No relay.json found. Run relayInit first.` (root not discovered)
- `repo "<repo>" not registered. Run relaySave in that repo first.` (repo absent)

### 3.5 `!relayResume`

**Tool call**: `relayResume` (args: `repo?, kind?`)

**Return**: rendered resume (same text as legacy plugin). Also writes `RESUME.md` at the relay root.

**Failure cases**:
- `No active baton set and no repo given. Run relaySwitch <repo> first.`
- `repo "<target>" not registered.` (when repo specified but absent)

### 3.6 `!relayStatus`

**Tool call**: `relayStatus`

**Return** (multiline status report):

```
Relay root: <root>
Project: <project_context | "(unset)">
Active baton: <active_baton | "(none)">
Repos (<n>):
  - <name> [<active_phase | "?">] conf=<confidence_score>[ · <n> handoff(s)][ → <next_session_starter, first 60 chars>]
Specs: <names, comma-joined | "(none)">
Drift: <spec (added), ...> | "(none)"
Updated: <updated_at>
```

### 3.7 `!relayAdd <file>`

**Tool call**: `relayAdd` (args: `file`, `repo?`)

**Return**:

```
Added handoff "<basename(file)>" to "<repo>".
Parsed <n> line(s) into open_threads.
Total open threads: <total>
```

**File path**: `<file>` is resolved relative to the current working directory of the agent session, not the Graphify workspace root.

## 4. Notes on state discovery and coordination

- **Root resolution** (see `PROTOCOL.md`) is **cached** in the plugin:
  - Named-repo ops (any tool with `repo` argument) resolve purely against the registry (`repos[name].path`).
  - No-repo ops (e.g., `relayStatus`) use the root cached at `bind` time.
  - The only write-type walk is `relayInit`; all other operations are in-memory lookups.
- **Fail-fast**: any no-repo tool with no cached root returns
  `No relay.json found. Run relayInit first.`
- **Tool ownership**: relay state (`relay.json`, `specs/`, `RESUME.md`, `next_step.md`) is **owned** by the plugin; the skill never touches the filesystem.

## 5. Optional context expansion (integration points)

The skill does not enforce any requirement to rebuild session context with other MCP servers. If a user wants to augment the session with document knowledge (`opendoc-mcp`) or code structure graphs (`graphify`), they can:

1. **opendoc-mcp**: call its `search` or `read` tools directly inside the agent session to retrieve and inline relevant specs, meeting the plugin’s emphasis on documentation-first.

2. **graphify**: invoke its `query_graph` or `trace_path` tools to explore codebase dependencies or call chains related to a given symbol (e.g., find all callers of `relaySave`).

These are *optional* and *out-of-band* — the skill remains thin and single-purpose. The agent can interleave them as needed without being forced.

## 6. Error handling

- All tool results are returned as-is; the skill does not post-process.
- In MCP, a tool result with `isError: true` still becomes the agent response (preserving legacy error texts).
- If GraphifyMCP or the plugin fails to load, the skill commands fail with a generic MCP error; the user can restart the Graphify session.

## 7. Privacy & deployment notes

- No internal network topologies, private hostnames, or credentials appear in the skill or plugin artifacts.
- All file paths are relative to the project root and never contain absolute hostnames or secrets.
- The plugin reads only under the workspace root; there are no global system scans.

## 8. References

- Relay tool contract: see `PROTOCOL.md`.
- Root discovery and caching semantics: see `openspec/changes/rust-mcp-migration/specs/stateful-cache/spec.md`.
