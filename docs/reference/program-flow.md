---
summary: "Complete program flow from shell invocation to running Gateway with active chat channels"
read_when:
  - Understanding the full bootstrap sequence
  - Debugging startup issues or module loading
  - Working on CLI commands, plugin loading, or gateway internals
  - Onboarding to the codebase
title: "Program Flow & Internals"
---

# Program flow & internals

Last updated: 2026-02-14

This document traces the entire execution path of OpenClaw — from the shell
command `openclaw` to a running Gateway instance with active chat channels.
It serves as an architecture reference for contributors and anyone debugging
startup behavior.

---

## 1. Shell entry point

```
~/.local/bin/openclaw          (bash wrapper)
  exec node dist/entry.js "$@"
```

The installed `openclaw` binary is a thin bash script that forwards all
arguments to the bundled Node.js entry point.

## 2. entry.js — bootstrap

**Path:** `dist/entry.js` (bundled from 8 source files, ~48 KB)

### Bundled source modules

| Module                     | Purpose                                                                 |
| -------------------------- | ----------------------------------------------------------------------- |
| `src/infra/home-dir.ts`    | Resolve home directory (OPENCLAW_HOME, HOME, USERPROFILE)               |
| `src/cli/profile-utils.ts` | Validate profile names (regex: `[a-z0-9_-]{1,64}`)                      |
| `src/cli/profile.ts`       | Parse `--profile` / `--dev` from argv                                   |
| `src/plugins/runtime.ts`   | Global plugin registry via `Symbol.for("openclaw.pluginRegistryState")` |
| `src/channels/registry.ts` | Channel metadata (8 core channels + aliases)                            |
| `src/config/paths.ts`      | Path resolution: config, state, OAuth, gateway lock                     |
| `src/logging/`             | File logger (tslog to JSON), console capture, subsystem loggers         |
| `src/terminal/`            | Color palette (Lobster theme), ANSI strip, terminal restore             |

### Bootstrap sequence

```
1. process.title = "openclaw"
2. installProcessWarningFilter()     // suppress punycode/SQLite warnings
3. normalizeEnv()                    // normalize ZAI_API_KEY
4. process --no-color flag
5. ensureExperimentalWarningSuppressed()
   // if needed: self-respawn as child with --disable-warning=ExperimentalWarning
6. parseCliProfileArgs(argv)         // extract --profile / --dev
7. applyCliProfileEnv()              // set OPENCLAW_PROFILE, STATE_DIR, CONFIG_PATH
8. import("./run-main-clWS7Jyh.js") // hand off to CLI orchestrator
```

The self-respawn in step 5 is notable: if the current process was not started
with the `--disable-warning=ExperimentalWarning` flag, entry.js re-executes
itself as a child process with that flag to suppress Node.js experimental
feature warnings.

## 3. run-main — CLI orchestrator

**Path:** `dist/run-main-clWS7Jyh.js` (~6.8 KB, imports ~80 modules)

Core flow of `runCli()`:

```
 1. stripWindowsNodeExec(argv)
 2. loadDotEnv({ quiet: true })        // load .env files
 3. normalizeEnv()
 4. ensureOpenClawCliOnPath()           // ensure CLI is on PATH
 5. assertSupportedRuntime()            // Node >= 22.12.0
 6. tryRouteCli(argv)                   // fast-path for routed commands
    // findRoutedCommand() from config-guard-CLQRyXpa.js
    // if match: execute immediately, return
 7. enableConsoleCapture()              // patch console.* to file logger
 8. buildProgram()                      // create Commander.js program
    // from program-BkzmONw5.js
 9. installUnhandledRejectionHandler()
10. getPrimaryCommand(argv)             // determine primary command
11. registerSubCliByName(program, cmd)  // lazy-load the command
12. registerPluginCliCommands(program)  // register plugin CLI commands
13. program.parseAsync(argv)            // execute the command
```

### Key imports

| Module       | File                           | Purpose                                      |
| ------------ | ------------------------------ | -------------------------------------------- |
| Config       | `config-DIhRWqvz.js` (200 KB)  | Config loading, VERSION constant, Zod schema |
| Config Guard | `config-guard-CLQRyXpa.js`     | Routed commands, CLI banner                  |
| Program      | `program-BkzmONw5.js`          | Commander program builder                    |
| SubCLIs      | `register.subclis-cfiNvW7L.js` | 25 lazy-loaded commands                      |
| Plugin CLI   | `cli-DQgjbT8B.js`              | Plugin command registration                  |
| Path Env     | `path-env-CXWUFfFv.js`         | Ensure CLI on PATH                           |
| Daemon RT    | `daemon-runtime-odoEo1Jb.js`   | Runtime compatibility                        |
| Loader       | `loader-Ca5F0s0m.js` (2.4 MB)  | Session/workspace logic                      |

## 4. Lazy-loaded SubCLI commands (25 modules)

Commands are imported only when invoked (lazy-loading pattern via
`registerSubCliByName`):

| Command      | Module                             | Purpose                     |
| ------------ | ---------------------------------- | --------------------------- |
| `gateway`    | `gateway-cli-Bx0hHZ_r.js` (653 KB) | Start/stop the Gateway      |
| `daemon`     | `daemon-cli-DcJvwF9p.js`           | Gateway as service (legacy) |
| `tui`        | `tui-cli-RAdHt_yw.js`              | Terminal UI                 |
| `models`     | `models-cli-hGDr_eKa.js` (96 KB)   | Model configuration         |
| `channels`   | `channels-cli-KhIZf06i.js`         | Channel management          |
| `plugins`    | `plugins-cli-CXjpHh8m.js`          | Plugin management           |
| `skills`     | `skills-cli-BOPQkeuy.js`           | Skills management           |
| `nodes`      | `nodes-cli-Dc05OLlw.js`            | Remote node commands        |
| `devices`    | `devices-cli-CV7z7qLX.js`          | Device pairing              |
| `acp`        | `acp-cli-DaMTmUaV.js`              | Agent Control Protocol      |
| `sandbox`    | `sandbox-cli-CUc83jPa.js` (115 KB) | Sandbox tools               |
| `cron`       | `cron-cli-CWc5iZrO.js`             | Cron scheduler              |
| `hooks`      | `hooks-cli-DBjLuANP.js`            | Hook management             |
| `webhooks`   | `webhooks-cli-BTL_YeQv.js`         | Webhook helpers             |
| `logs`       | `logs-cli-Cd0-WlGZ.js`             | Gateway logs                |
| `system`     | `system-cli-BfDkGCOI.js`           | System events               |
| `dns`        | `dns-cli-Dn3pI4Pd.js`              | DNS helpers                 |
| `docs`       | `docs-cli-DcwaNAax.js`             | Documentation               |
| `pairing`    | `pairing-cli-DtDEO_DC.js`          | Pairing helpers             |
| `directory`  | `directory-cli-BGor4p8t.js`        | Directory commands          |
| `security`   | `security-cli-BYIH_WyU.js`         | Security helpers            |
| `update`     | `update-cli-D1RW9MqH.js`           | CLI update                  |
| `completion` | `completion-cli-BxuJu_TM.js`       | Shell completion            |
| `node`       | `node-cli-CzHdEENi.js`             | Node control                |
| `approvals`  | `exec-approvals-cli-B_Unb0rw.js`   | Exec approvals              |

## 5. Gateway startup (primary path: `openclaw gateway start`)

```
gateway-cli-Bx0hHZ_r.js
  loadConfig()
  resolveStateDir() / resolveOAuthDir()
  daemon-cli-DcJvwF9p.js  -->  spawn Gateway process
      server-startup.ts:
          1. loadGatewayPlugins()        // fill plugin registry
          2. Start channel monitors      // Telegram, Discord, WhatsApp, ...
          3. Start sidecars:
                Browser Control Server
                Gmail Watcher
                Hook Handler
                Plugin Services
                Memory Backend (LanceDB)
          4. Bind WebSocket server on port 18789
          5. Start HTTP server:
                Control UI (web dashboard)
                Canvas / A2UI
                Webhook endpoints
                OpenAI-compatible API
                Tool invocation
          6. Start Cron service
          7. Start Discovery service
```

The Gateway runs as a foreground process (or daemonized via systemd/launchd).
Port `18789` is the default for both WS and HTTP on the same server.

## 6. Plugin system

**Source:** `src/plugins/`

### Loading sequence

```
loadOpenClawPlugins()
  1. discoverOpenClawPlugins()     // scan extensions/, workspace, global
  2. loadPluginManifestRegistry()  // read each plugin's package.json
  3. Jiti (JIT transpiler)         // dynamic import of TypeScript directly
  4. plugin.register(api)          // plugin registers itself via the API
```

### Plugin API (`OpenClawPluginApi`)

```
api.registerChannel()         // chat channel
api.registerTool()            // agent tool
api.registerHook()            // event hook
api.registerCommand()         // agent command
api.registerGatewayHandler()  // gateway RPC method
api.registerHttpHandler()     // HTTP endpoint
api.registerHttpRoute()       // HTTP route
api.registerProvider()        // media provider
api.registerService()         // long-running service
api.registerCliCommand()      // CLI command
```

### Registry structure

```
plugins[], tools[], hooks[], typedHooks[], channels[],
providers[], gatewayHandlers{}, httpHandlers[], httpRoutes[],
cliRegistrars[], services[], commands[], diagnostics[]
```

## 7. Channel architecture

**36 extensions** in `extensions/`:

### Messaging channels

telegram, discord, whatsapp, slack, signal, irc, matrix, line,
googlechat, feishu, mattermost, nextcloud-talk, imessage,
bluebubbles, tlon, twitch, zalo, zalouser, nostr

### Channel plugin interface (simplified)

```
ChannelPlugin {
  id, meta, capabilities,
  config, setup?,                    // configuration
  auth?, security?, pairing?,        // authentication
  messaging?, outbound?, streaming?, // messages
  threading?, groups?, mentions?,    // conversations
  status?, heartbeat?,               // monitoring
  gateway?, gatewayMethods?,         // gateway integration
  commands?, actions?,               // commands
  agentTools?, agentPrompt?          // agent integration
}
```

### Core libraries per channel

| Channel  | Library                   |
| -------- | ------------------------- |
| Telegram | `grammy`                  |
| WhatsApp | `@whiskeysockets/baileys` |
| Discord  | internal dist module      |
| Slack    | `@slack/bolt`             |
| LINE     | `@line/bot-sdk`           |
| Feishu   | `@larksuiteoapi/node-sdk` |

## 8. Gateway RPC methods (~90 methods)

Core categories:

- **`health`** — health check
- **`config.*`** — get, set, apply, patch
- **`agents.*`** — list, create, update, delete
- **`send` / `chat.*`** — send messages, history, abort
- **`sessions.*`** — list, preview, patch, reset
- **`models.list`** — available models
- **`cron.*`** — scheduled tasks
- **`skills.*`** — skill management
- **`node.*` / `device.*`** — remote nodes and devices
- **`channels.status` / `channels.logout`**
- **`tts.*`** — text-to-speech
- **`exec.approval.*`** — execution approvals
- **`logs.tail`** — log streaming
- **`usage.*`** — usage statistics

## 9. Heavy chunks (core logic)

| Chunk                       | Size   | Contents                            |
| --------------------------- | ------ | ----------------------------------- |
| `loader-Ca5F0s0m.js`        | 2.4 MB | Session management, workspace logic |
| `reply-CRgPtCO_.js`         | 2.4 MB | Agent response processing           |
| `pi-embedded-Dao7qsb5.js`   | 2.4 MB | Embedded agent runtime              |
| `gateway-cli-Bx0hHZ_r.js`   | 653 KB | Gateway command set                 |
| `config-DIhRWqvz.js`        | 200 KB | Config system + Zod schema          |
| `sandbox-cli-CUc83jPa.js`   | 115 KB | Sandbox environment                 |
| `auth-profiles-BZAsA3O9.js` | 104 KB | Auth system                         |

## 10. Dependency graph (simplified)

```
                        +-------------+
                        |    Shell    |
                        |  openclaw   |
                        +------+------+
                               |
                        +------v------+
                        |  entry.js   |  Bootstrap: env, logging, profile
                        +------+------+
                               |
                     +---------v---------+
                     |  run-main-*.js    |  CLI orchestrator
                     +---------+---------+
                               |
              +----------------+----------------+
              |                |                |
      +-------v------+  +-----v-----+  +-------v-------+
      | config-*.js  |  | program-  |  | register.     |
      | Config+Zod   |  | *.js      |  | subclis-*.js  |
      +-------+------+  | Commander |  | 25 commands   |
              |         +-----+-----+  +-------+-------+
              |               |                |
              +---------------+----------------+
                              | (on "gateway start")
                    +---------v----------+
                    |  gateway-cli-*.js  |  Gateway commands
                    +---------+----------+
                              |
                    +---------v----------+
                    |  Gateway server    |  WS + HTTP on :18789
                    +---------+----------+
                              |
         +--------------------+--------------------+
         |                    |                    |
   +-----v-----+      +------v------+      +------v------+
   |  Plugins  |      |  Channels   |      |  Services   |
   |  36 ext.  |      |  19 platf.  |      |  Cron, TTS  |
   +-----------+      +-------------+      |  Memory,    |
                                           |  Browser    |
                                           +-------------+
```

## 11. Configuration & filesystem

```
~/.openclaw/
  openclaw.json          // main config (JSON5)
  credentials/oauth.json // OAuth tokens
  agents/                // agent state
  workspace/             // agent working directory
  browser/               // browser automation
  canvas/                // canvas artifacts
  cron/                  // cron state
  devices/               // device pairing
  extensions/            // extension state
  identity/              // agent identity
  media/                 // media cache
  subagents/             // sub-agent configs
  whisper-env/           // speech-to-text

/tmp/openclaw/
  openclaw-YYYY-MM-DD.log  // rolling logs (24h retention)
```

## 12. Project metadata

- **Name:** openclaw
- **Version:** 2026.2.10
- **License:** MIT
- **Node:** >= 22.12.0
- **Package manager:** pnpm 10.23.0
- **Build:** tsdown / Rolldown
- **Tests:** vitest
- **Dependencies:** 52 npm packages, 490+ bundle files in dist/
- **Platforms:** Linux, Docker, macOS, iOS, Android, Nix
