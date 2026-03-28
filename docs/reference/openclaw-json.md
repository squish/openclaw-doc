---
title: "openclaw.json Reference"
summary: "Comprehensive field-by-field reference for the openclaw.json configuration file"
read_when:
  - You need a complete overview of every configuration section in one place
  - You are setting up openclaw.json for the first time
  - You want to understand the full structure and all top-level keys
---

# openclaw.json Reference

`~/.openclaw/openclaw.json` is the main configuration file for OpenClaw. It is read at startup and watched for live changes.

**Format:** JSON5 (comments and trailing commas allowed). Regular JSON also works.

**Validation:** Strict. Unknown keys and malformed types prevent gateway startup. The only root-level exception is `$schema` (string), so editors can attach JSON Schema metadata.

**All fields are optional.** OpenClaw uses safe defaults when the file is missing or a field is omitted.

For task-oriented setup help, see [Configuration](/gateway/configuration).
For ready-to-use examples, see [Configuration Examples](/gateway/configuration-examples).

---

## Top-level structure

```json5
{
  // Environment variable injection
  env: { ... },

  // Auth profiles (OAuth, API keys)
  auth: { ... },

  // Agent identity (name, theme, emoji)
  // Shorthand; full form is agents.list[].identity
  identity: { ... },

  // Logging configuration
  logging: { ... },

  // CLI banner behavior
  cli: { ... },

  // Channel integrations (WhatsApp, Telegram, Discord, etc.)
  channels: { ... },

  // WhatsApp Baileys connection settings (sibling of channels)
  web: { ... },

  // Agent defaults and per-agent overrides
  agents: { ... },

  // Session scoping, resets, and store settings
  session: { ... },

  // Message formatting, queue, TTS
  messages: { ... },

  // Talk mode (macOS/iOS/Android voice)
  talk: { ... },

  // Tool profiles, allow/deny, elevated access, media
  tools: { ... },

  // Custom model providers and base URLs
  models: { ... },

  // Skills configuration
  skills: { ... },

  // Plugin configuration
  plugins: { ... },

  // Browser automation (CDP)
  browser: { ... },

  // Control UI customization
  ui: { ... },

  // Gateway networking, auth, and port
  gateway: { ... },

  // Webhook ingress mappings
  hooks: { ... },

  // Canvas host for agent-editable HTML/JS
  canvasHost: { ... },

  // mDNS / DNS-SD service discovery
  discovery: { ... },

  // Secrets provider configuration
  secrets: { ... },

  // Cron job defaults
  cron: { ... },

  // Multi-agent routing bindings
  bindings: [ ... ],

  // Chat command handling (text commands, bash, /config, etc.)
  commands: { ... },

  // Wizard metadata (written by guided flows)
  wizard: { ... },
}
```

---

## `env` — environment variables

Inject environment variables that are applied before any channel or model config is evaluated. Inline vars are only applied when the process environment is missing the key.

```json5
{
  env: {
    // Top-level keys are treated as direct env vars
    OPENROUTER_API_KEY: "sk-or-...",

    // vars sub-object is equivalent
    vars: {
      GROQ_API_KEY: "gsk-...",
    },

    // Import missing expected keys from your login shell profile
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

**Env var substitution:** Reference vars in any config string with `${VAR_NAME}`. Only uppercase `[A-Z_][A-Z0-9_]*` patterns match. Missing or empty vars throw an error at config load. Escape with `$${VAR}` for a literal `${VAR}`.

`.env` files in CWD and `~/.openclaw/.env` are auto-loaded (neither overrides existing vars).

See [Environment](/help/environment).

---

## `auth` — authentication profiles

Stores OAuth and API key profile metadata. Credential values (tokens, keys) are stored in `<agentDir>/auth-profiles.json`, not here.

```json5
{
  auth: {
    profiles: {
      "anthropic:me@example.com": {
        provider: "anthropic",
        mode: "oauth",
        email: "me@example.com",
      },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai:default": { provider: "openai", mode: "api_key" },
    },
    order: {
      anthropic: ["anthropic:me@example.com", "anthropic:work"],
      openai: ["openai:default"],
    },
  },
}
```

- `auth.profiles`: named provider profiles. Each entry references a provider + auth mode.
- `auth.order`: provider-level rotation order. First entry is tried first; failover proceeds down the list.
- Per-agent profiles are stored at `<agentDir>/auth-profiles.json`.
- `auth-profiles.json` supports `keyRef` and `tokenRef` for SecretRef-backed credentials.

See [OAuth](/concepts/oauth) and [Secrets Management](/gateway/secrets).

---

## `identity` — agent identity (shorthand)

Shorthand root-level identity block, equivalent to setting `agents.list[0].identity`. Written by the macOS onboarding assistant.

```json5
{
  identity: {
    name: "Samantha",
    theme: "helpful sloth",
    emoji: "🦥",
    avatar: "avatars/samantha.png", // workspace-relative path, http(s) URL, or data: URI
  },
}
```

Derived defaults:
- `messages.ackReaction` inherits from `identity.emoji` (falls back to 👀)
- `mentionPatterns` derived from `identity.name` and `identity.emoji`

---

## `logging` — logging configuration

```json5
{
  logging: {
    level: "info",                    // trace | debug | info | warn | error
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty",           // pretty | compact | json
    redactSensitive: "tools",         // off | tools
    redactPatterns: ["\\bTOKEN\\b\\s*[=:]\\s*([\"']?)([^\\s\"']+)\\1"],
  },
}
```

- Default log file: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`. Set `logging.file` for a stable path.
- `consoleLevel` is bumped to `debug` when `--verbose` is passed.
- `redactSensitive: "tools"` redacts tool call arguments containing credential patterns from logs.

---

## `cli` — CLI behavior

```json5
{
  cli: {
    banner: {
      taglineMode: "random", // random | default | off
    },
  },
}
```

- `"random"` (default): rotating funny/seasonal taglines.
- `"default"`: fixed neutral tagline (`All your chats, one OpenClaw.`).
- `"off"`: no tagline text (banner title/version still shown).
- Set env `OPENCLAW_HIDE_BANNER=1` to suppress the entire banner.

---

## `channels` — messaging platform integrations

Channels start automatically when their config section exists (unless `enabled: false`). Each channel uses DM and group policies for access control.

### DM and group access policies

| DM policy           | Behavior                                                        |
| ------------------- | --------------------------------------------------------------- |
| `pairing` (default) | Unknown senders get a one-time pairing code; owner must approve |
| `allowlist`         | Only senders in `allowFrom` (or paired allow store)             |
| `open`              | Allow all inbound DMs (requires `allowFrom: ["*"]`)             |
| `disabled`          | Ignore all inbound DMs                                          |

| Group policy          | Behavior                                               |
| --------------------- | ------------------------------------------------------ |
| `allowlist` (default) | Only groups in the configured allowlist                |
| `open`                | Bypass group allowlists (mention-gating still applies) |
| `disabled`            | Block all group/room messages                          |

### `channels.defaults`

Shared settings applied across all channel providers:

```json5
{
  channels: {
    defaults: {
      groupPolicy: "allowlist", // open | allowlist | disabled
      heartbeat: {
        showOk: false,
        showAlerts: true,
        useIndicator: true,
      },
    },
  },
}
```

### `channels.modelByChannel`

Pin specific channel IDs to a model:

```json5
{
  channels: {
    modelByChannel: {
      discord: { "123456789012345678": "anthropic/claude-opus-4-6" },
      slack: { C1234567890: "openai/gpt-4.1" },
      telegram: {
        "-1001234567890": "openai/gpt-4.1-mini",
        "-1001234567890:topic:99": "anthropic/claude-sonnet-4-6",
      },
    },
  },
}
```

### `channels.whatsapp`

WhatsApp via Baileys Web. Starts automatically when a linked session exists.

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",           // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123"],
      textChunkLimit: 4000,
      chunkMode: "length",           // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true,
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
      groups: { "*": { requireMention: true } },
      // Multi-account:
      accounts: {
        default: {},
        personal: {},
        biz: {},
      },
    },
  },
  web: {
    enabled: true,
    heartbeatSeconds: 60,
    reconnect: {
      initialMs: 2000,
      maxMs: 120000,
      factor: 1.4,
      jitter: 0.2,
      maxAttempts: 0,
    },
  },
}
```

See [WhatsApp](/channels/whatsapp).

### `channels.telegram`

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "your-bot-token",     // or TELEGRAM_BOT_TOKEN env
      dmPolicy: "pairing",
      allowFrom: ["tg:123456789"],
      groupPolicy: "allowlist",
      groups: {
        "*": { requireMention: true },
        "-1001234567890": {
          allowFrom: ["@admin"],
          systemPrompt: "Keep answers brief.",
          topics: {
            "99": { requireMention: false, skills: ["search"], systemPrompt: "Stay on topic." },
          },
        },
      },
      customCommands: [
        { command: "backup", description: "Git backup" },
      ],
      historyLimit: 50,
      replyToMode: "first",           // off | first | all
      linkPreview: true,
      streaming: "partial",           // off | partial | block | progress
      actions: { reactions: true, sendMessage: true },
      reactionNotifications: "own",   // off | own | all | allowlist
      mediaMaxMb: 100,
      retry: {
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
      network: {
        autoSelectFamily: true,
        dnsResultOrder: "ipv4first",
      },
      proxy: "socks5://localhost:9050",
      webhookUrl: "https://example.com/telegram-webhook",
      webhookSecret: "secret",
      webhookPath: "/telegram-webhook",
    },
  },
}
```

See [Telegram](/channels/telegram).

### `channels.discord`

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",        // or DISCORD_BOT_TOKEN env
      mediaMaxMb: 8,
      allowBots: false,
      actions: {
        reactions: true, stickers: true, polls: true, permissions: true,
        messages: true, threads: true, pins: true, search: true,
        memberInfo: true, roleInfo: true, roles: false, channelInfo: true,
        voiceStatus: true, events: true, moderation: false,
      },
      replyToMode: "off",
      dmPolicy: "pairing",
      allowFrom: ["1234567890"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["openclaw-dm"] },
      guilds: {
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          ignoreOtherMentions: true,
          reactionNotifications: "own",
          users: ["987654321098765432"],
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["987654321098765432"],
              skills: ["docs"],
              systemPrompt: "Short answers only.",
            },
          },
        },
      },
      historyLimit: 20,
      textChunkLimit: 2000,
      chunkMode: "length",            // length | newline
      streaming: "off",
      maxLinesPerMessage: 17,
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSubagentSessions: false,
      },
      voice: {
        enabled: true,
        autoJoin: [{ guildId: "123456789012345678", channelId: "234567890123456789" }],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        tts: { provider: "openai", openai: { voice: "alloy" } },
      },
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

See [Discord](/channels/discord).

### `channels.slack`

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",           // or SLACK_BOT_TOKEN env
      appToken: "xapp-...",           // or SLACK_APP_TOKEN env (socket mode)
      dmPolicy: "pairing",
      allowFrom: ["U123"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["G123"] },
      channels: {
        C123: { allow: true, requireMention: true, allowBots: false },
        "#general": {
          allow: true,
          requireMention: true,
          allowBots: false,
          users: ["U123"],
          skills: ["docs"],
          systemPrompt: "Short answers only.",
        },
      },
      historyLimit: 50,
      allowBots: false,
      reactionNotifications: "own",
      reactionAllowlist: ["U123"],
      replyToMode: "off",             // off | first | all
      thread: { historyScope: "thread", inheritParent: false },
      actions: {
        reactions: true, messages: true, pins: true, memberInfo: true, emojiList: true,
      },
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
      typingReaction: "hourglass_flowing_sand",
      textChunkLimit: 4000,
      chunkMode: "length",
      streaming: "partial",
      nativeStreaming: true,
      mediaMaxMb: 20,
    },
  },
}
```

See [Slack](/channels/slack).

### `channels.signal`

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123",
      dmPolicy: "pairing",
      allowFrom: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      configWrites: true,
      reactionNotifications: "own",   // off | own | all | allowlist
      reactionAllowlist: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      historyLimit: 50,
    },
  },
}
```

See [Signal](/channels/signal).

### `channels.googlechat`

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url",
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890",
      dm: { enabled: true, policy: "pairing", allowFrom: ["users/1234567890"] },
      groupPolicy: "allowlist",
      groups: { "spaces/AAAA": { allow: true, requireMention: true } },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

See [Google Chat](/channels/googlechat).

### `channels.bluebubbles`

BlueBubbles is the recommended iMessage path (plugin-backed).

```json5
{
  channels: {
    bluebubbles: {
      enabled: true,
      dmPolicy: "pairing",
      // serverUrl, password, webhookPath, etc.: see /channels/bluebubbles
    },
  },
}
```

See [BlueBubbles](/channels/bluebubbles).

### `channels.imessage`

Legacy iMessage via `imsg rpc` (stdio). Prefer BlueBubbles for new setups.

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
      remoteHost: "user@gateway-host",
      dmPolicy: "pairing",
      allowFrom: ["+15555550123", "user@example.com", "chat_id:123"],
      historyLimit: 50,
      includeAttachments: false,
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      mediaMaxMb: 16,
      service: "auto",
      region: "US",
    },
  },
}
```

See [iMessage](/channels/imessage).

### `channels.msteams`

Microsoft Teams (extension-backed).

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId, appPassword, tenantId, etc.: see /channels/msteams
    },
  },
}
```

See [Microsoft Teams](/channels/msteams).

### `channels.mattermost`

Mattermost (plugin-backed: `openclaw plugins install @openclaw/mattermost`).

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
      chatmode: "oncall",             // oncall | onmessage | onchar
      oncharPrefixes: [">", "!"],
      commands: {
        native: true,
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      chunkMode: "length",
    },
  },
}
```

See [Mattermost](/channels/mattermost).

### `channels.irc`

IRC (extension-backed).

```json5
{
  channels: {
    irc: {
      enabled: true,
      dmPolicy: "pairing",
      configWrites: true,
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "${IRC_NICKSERV_PASSWORD}",
        register: false,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

See [IRC](/channels/irc).

### Multi-account channels

Run multiple accounts per channel by using an `accounts` map:

```json5
{
  channels: {
    telegram: {
      accounts: {
        default: { name: "Primary bot", botToken: "123456:ABC..." },
        alerts:  { name: "Alerts bot",  botToken: "987654:XYZ..." },
      },
    },
  },
}
```

- `default` account is used when `accountId` is omitted.
- Base channel settings apply to all accounts unless overridden per account.
- Use `bindings[].match.accountId` to route each account to a different agent.

### `commands` — chat command handling

```json5
{
  commands: {
    native: "auto",    // register native commands when supported
    text: true,        // parse /commands in chat messages
    bash: false,       // allow ! (alias: /bash)
    bashForegroundMs: 2000,
    config: false,     // allow /config
    debug: false,      // allow /debug
    restart: false,    // allow /restart + gateway restart tool
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

- `bash: true` requires `tools.elevated.enabled` and sender in `tools.elevated.allowFrom`.
- `config: true` enables `/config` reads and writes to `openclaw.json`.

---

## `agents` — agent defaults and per-agent overrides

### `agents.defaults`

Base settings applied to all agents unless overridden.

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      repoRoot: "~/Projects/openclaw",    // optional; auto-detected when omitted
      skipBootstrap: false,
      bootstrapMaxChars: 20000,
      bootstrapTotalMaxChars: 150000,
      bootstrapPromptTruncationWarning: "once", // off | once | always
      imageMaxDimensionPx: 1200,
      userTimezone: "America/Chicago",
      timeFormat: "auto",                 // auto | 12 | 24
    },
  },
}
```

### `agents.defaults.model`

Primary model, fallbacks, and model catalog:

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6":    { alias: "opus" },
        "anthropic/claude-sonnet-4-6":  { alias: "sonnet" },
        "openai/gpt-5.4":               { alias: "gpt" },
        "openai/gpt-5-mini":            { alias: "gpt-mini" },
      },
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["openai/gpt-5.2"],
      },
      imageModel: {
        primary: "openrouter/qwen/qwen-2.5-vl-72b-instruct:free",
      },
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
        fallbacks: ["google/gemini-3.1-flash-image-preview"],
      },
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5-mini"],
      },
      pdfMaxBytesMb: 10,
      pdfMaxPages: 20,
      thinkingDefault: "low",           // off | minimal | low | medium | high | xhigh | adaptive
      verboseDefault: "off",
      elevatedDefault: "on",
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      contextTokens: 200000,
      maxConcurrent: 1,
    },
  },
}
```

- `model` accepts a string (`"provider/model"`) or `{ primary, fallbacks }`.
- `models` defines the catalog and allowlist for `/model` in chat.
- `maxConcurrent`: max parallel agent runs across sessions (each session serialized).

### `agents.defaults.cliBackends`

Optional text-only fallback backends when API providers fail:

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "claude-cli": { command: "/opt/homebrew/bin/claude" },
        "my-cli": {
          command: "my-cli",
          args: ["--json"],
          output: "json",
          modelArg: "--model",
          sessionArg: "--session",
          sessionMode: "existing",
          systemPromptArg: "--system",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
        },
      },
    },
  },
}
```

### `agents.defaults.heartbeat`

Periodic heartbeat runs:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",             // 0m disables; supports ms/s/m/h
        model: "openai/gpt-5.2-mini",
        includeReasoning: false,
        lightContext: false,      // keep only HEARTBEAT.md from bootstrap files
        isolatedSession: false,   // each heartbeat in fresh session (lower token cost)
        session: "main",
        to: "+15555550123",
        directPolicy: "allow",    // allow | block
        target: "none",           // none | last | whatsapp | telegram | discord | ...
        prompt: "Read HEARTBEAT.md if it exists...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
      },
    },
  },
}
```

### `agents.defaults.compaction`

Context compaction behavior:

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard",        // default | safeguard (chunked summarization for long histories)
        timeoutSeconds: 900,
        reserveTokensFloor: 24000,
        identifierPolicy: "strict", // strict | off | custom
        identifierInstructions: "Preserve deployment IDs...",
        postCompactionSections: ["Session Startup", "Red Lines"],
        model: "openrouter/anthropic/claude-sonnet-4-6",
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 6000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md...",
        },
      },
    },
  },
}
```

See [Compaction](/concepts/compaction).

### `agents.defaults.contextPruning`

Prune old tool results from in-memory context before sending to the LLM. Does **not** modify session history on disk.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl",        // off | cache-ttl
        ttl: "1h",
        keepLastAssistants: 3,
        softTrimRatio: 0.3,
        hardClearRatio: 0.5,
        minPrunableToolChars: 50000,
        softTrim: { maxChars: 4000, headChars: 1500, tailChars: 1500 },
        hardClear: { enabled: true, placeholder: "[Old tool result content cleared]" },
        tools: { deny: ["browser", "canvas"] },
      },
    },
  },
}
```

See [Session Pruning](/concepts/session-pruning).

### `agents.defaults.sandbox`

Sandbox configuration for running agent sessions in isolation:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",         // off | non-main | all
        backend: "docker",        // docker | ssh | openshell
        scope: "agent",           // session | agent | shared
        workspaceAccess: "none",  // none | ro | rw
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          containerPrefix: "openclaw-sbx-",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git curl jq",
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
          dns: ["1.1.1.1", "8.8.8.8"],
          extraHosts: ["internal.service:10.0.0.5"],
          binds: ["/home/user/source:/source:rw"],
        },
        ssh: {
          target: "user@gateway-host:22",
          command: "ssh",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // SecretRef alternatives:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
        },
        browser: {
          enabled: false,
          image: "openclaw-sandbox-browser:bookworm-slim",
          network: "openclaw-sandbox-browser",
          cdpPort: 9222,
          cdpSourceRange: "172.21.0.1/32",
          vncPort: 5900,
          noVncPort: 6080,
          headless: false,
          enableNoVnc: true,
          allowHostControl: false,
          autoStart: true,
          autoStartTimeoutMs: 12000,
        },
        prune: {
          idleHours: 24,
          maxAgeDays: 7,
        },
      },
    },
  },
}
```

Build images: `scripts/sandbox-setup.sh` and `scripts/sandbox-browser-setup.sh`.

See [Sandboxing](/gateway/sandboxing).

### `agents.defaults` — streaming and typing

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off",   // on | off
      blockStreamingBreak: "text_end",// text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200 },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off | natural | custom
      typingMode: "instant",           // never | instant | thinking | message
      typingIntervalSeconds: 6,
    },
  },
}
```

See [Streaming](/concepts/streaming) and [Typing Indicators](/concepts/typing-indicators).

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 1,
        runTimeoutSeconds: 900,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model`: default model for spawned sub-agents. If omitted, sub-agents inherit the caller's model.
- `runTimeoutSeconds`: default timeout for `sessions_spawn` when the tool call omits it. `0` means no timeout.
- Per-subagent tool policy: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

### `agents.list` — per-agent overrides

Define multiple agents, each with its own workspace, model, and settings:

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        name: "Main Agent",
        workspace: "~/.openclaw/workspace",
        agentDir: "~/.openclaw/agents/main/agent",
        model: "anthropic/claude-opus-4-6",
        thinkingDefault: "high",
        reasoningDefault: "on",
        fastModeDefault: false,
        params: { cacheRetention: "none" },
        identity: {
          name: "Samantha",
          theme: "helpful sloth",
          emoji: "🦥",
          avatar: "avatars/samantha.png",
        },
        heartbeat: { every: "1h", target: "whatsapp" },
        sandbox: { mode: "off" },
        tools: {
          profile: "coding",
          allow: ["browser"],
          deny: ["canvas"],
          elevated: { enabled: true },
        },
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw"],
        },
        subagents: {
          allowAgents: ["*"], // ["*"] = any agent; default: same agent only
        },
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
    ],
  },
}
```

- `id`: stable agent id (required for named agents).
- `default`: first agent with `default: true` is the default agent.
- `model`: string sets primary only; `{ primary, fallbacks }` sets both.

---

## `bindings` — multi-agent routing

Route messages to specific agents based on channel, account, peer, or guild:

```json5
{
  bindings: [
    // Route by account
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },

    // Route by peer (direct chat or group)
    {
      agentId: "support",
      match: { channel: "telegram", peer: { kind: "group", id: "-1001234567890" } },
    },

    // ACP persistent binding
    {
      type: "acp",
      agentId: "coder",
      match: { channel: "discord", peer: { kind: "channel", id: "channel:123456789" } },
      acp: { mode: "run", label: "coder" },
    },
  ],
}
```

**Match field precedence (deterministic):**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (exact)
5. `match.accountId: "*"` (channel-wide)
6. Default agent

Within each tier, the first matching `bindings` entry wins.

See [Multi-Agent](/concepts/multi-agent) and [ACP Agents](/tools/acp-agents).

---

## `session` — session management

```json5
{
  session: {
    dmScope: "per-channel-peer", // main | per-peer | per-channel-peer | per-account-channel-peer
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"],
    },
    reset: {
      mode: "daily",      // daily | idle
      atHour: 4,
      idleMinutes: 60,
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      direct: { mode: "idle", idleMinutes: 240 },
      group:  { mode: "idle", idleMinutes: 120 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    parentForkMaxTokens: 100000,
    maintenance: {
      mode: "warn",         // warn | enforce
      pruneAfter: "30d",
      maxEntries: 500,
      rotateBytes: "10mb",
      resetArchiveRetention: "30d",
      maxDiskBytes: "500mb",
      highWaterBytes: "400mb",
    },
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
    },
    agentToAgent: { maxPingPongTurns: 5 },
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

**`dmScope` values:**
- `main`: all DMs share the main session
- `per-peer`: isolate by sender id across channels
- `per-channel-peer`: isolate per channel + sender (recommended for multi-user)
- `per-account-channel-peer`: isolate per account + channel + sender (recommended for multi-account)

See [Session Management](/concepts/session).

---

## `messages` — message formatting and queue

```json5
{
  messages: {
    responsePrefix: "🦞",            // or "auto" (derives from identity.name)
    ackReaction: "👀",
    ackReactionScope: "group-mentions", // group-mentions | group-all | direct | all
    removeAckAfterReply: false,
    groupChat: {
      historyLimit: 50,
    },
    queue: {
      mode: "collect",               // steer | followup | collect | steer-backlog | steer+backlog | queue | interrupt
      debounceMs: 1000,
      cap: 20,
      drop: "summarize",             // old | new | summarize
      byChannel: {
        whatsapp: "collect",
        telegram: "collect",
      },
    },
    inbound: {
      debounceMs: 2000,              // 0 disables; batches rapid messages into one turn
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
    tts: {
      auto: "always",                // off | always | inbound | tagged
      mode: "final",                 // final | all
      provider: "elevenlabs",
      summaryModel: "openai/gpt-4.1-mini",
      modelOverrides: { enabled: true },
      maxTextLength: 4000,
      timeoutMs: 30000,
      prefsPath: "~/.openclaw/settings/tts.json",
      elevenlabs: {
        apiKey: "elevenlabs_api_key",
        baseUrl: "https://api.elevenlabs.io",
        voiceId: "voice_id",
        modelId: "eleven_multilingual_v2",
        seed: 42,
        applyTextNormalization: "auto",
        languageCode: "en",
        voiceSettings: {
          stability: 0.5,
          similarityBoost: 0.75,
          style: 0.0,
          useSpeakerBoost: true,
          speed: 1.0,
        },
      },
      openai: {
        apiKey: "openai_api_key",
        baseUrl: "https://api.openai.com/v1",
        model: "gpt-4o-mini-tts",
        voice: "alloy",
      },
    },
  },
}
```

**Response prefix template variables:**

| Variable          | Example                     |
| ----------------- | --------------------------- |
| `{model}`         | `claude-opus-4-6`           |
| `{modelFull}`     | `anthropic/claude-opus-4-6` |
| `{provider}`      | `anthropic`                 |
| `{thinkingLevel}` | `high`, `low`, `off`        |
| `{identity.name}` | from agent identity         |

Variables are case-insensitive. `{think}` is an alias for `{thinkingLevel}`.

See [TTS](/tts).

---

## `talk` — Talk mode (macOS/iOS/Android voice)

```json5
{
  talk: {
    voiceId: "elevenlabs_voice_id",
    voiceAliases: {
      Clawd: "EXAVITQu4vr4xnSDxMaL",
      Roger: "CwhRBWXzGAHq8TQ4Fs17",
    },
    modelId: "eleven_v3",
    outputFormat: "mp3_44100_128",
    apiKey: "elevenlabs_api_key",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
  },
}
```

- `silenceTimeoutMs`: controls pause window before transcript is sent (platform defaults: 700ms macOS/Android, 900ms iOS).
- `voiceAliases` lets Talk directives use friendly names.

---

## `tools` — tool configuration

### Tool profiles

| Profile     | Includes                                                                           |
| ----------- | ---------------------------------------------------------------------------------- |
| `minimal`   | `session_status` only                                                              |
| `coding`    | `group:fs`, `group:runtime`, `group:sessions`, `group:memory`, `image`             |
| `messaging` | `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status` |
| `full`      | No restriction (same as unset)                                                     |

### Tool groups

| Group              | Tools                                                                                    |
| ------------------ | ---------------------------------------------------------------------------------------- |
| `group:runtime`    | `exec`, `process`                                                                        |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                   |
| `group:sessions`   | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `session_status` |
| `group:memory`     | `memory_search`, `memory_get`                                                            |
| `group:web`        | `web_search`, `web_fetch`                                                                |
| `group:ui`         | `browser`, `canvas`                                                                      |
| `group:automation` | `cron`, `gateway`                                                                        |
| `group:messaging`  | `message`                                                                                |
| `group:nodes`      | `nodes`                                                                                  |
| `group:openclaw`   | All built-in tools (excludes provider plugins)                                           |

### `tools` fields

```json5
{
  tools: {
    profile: "coding",             // minimal | coding | messaging | full
    allow: ["group:web"],
    deny: ["browser", "canvas"],

    // Per-provider tool restriction
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.2": { allow: ["group:fs"] },
    },

    // Elevated (host) exec access
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123"],
      },
    },

    // exec and process configuration
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      applyPatch: {
        enabled: false,
        allowModels: ["gpt-5.2"],
      },
    },

    // Loop detection (disabled by default)
    loopDetection: {
      enabled: true,
      historySize: 30,
      warningThreshold: 10,
      criticalThreshold: 20,
      globalCircuitBreakerThreshold: 30,
      detectors: {
        genericRepeat: true,
        knownPollNoProgress: true,
        pingPong: true,
      },
    },

    // Web search and fetch
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key",   // or BRAVE_API_KEY env
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        maxChars: 50000,
        maxCharsCap: 50000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        userAgent: "custom-ua",
      },
    },

    // Media understanding (audio/video/image)
    media: {
      concurrency: 2,
      audio: {
        enabled: true,
        maxBytes: 20971520,
        scope: {
          default: "deny",
          rules: [{ action: "allow", match: { chatType: "direct" } }],
        },
        models: [
          { provider: "openai", model: "gpt-4o-mini-transcribe" },
          { type: "cli", command: "whisper", args: ["--model", "base", "{{MediaPath}}"] },
        ],
      },
      video: {
        enabled: true,
        maxBytes: 52428800,
        models: [{ provider: "google", model: "gemini-3-flash-preview" }],
      },
    },

    // Session tool visibility
    sessions: {
      visibility: "tree",          // self | tree | agent | all
    },

    // sessions_spawn attachment support
    sessions_spawn: {
      attachments: {
        enabled: false,
        maxTotalBytes: 5242880,
        maxFiles: 50,
        maxFileBytes: 1048576,
        retainOnSessionKeep: false,
      },
    },

    // Cross-agent messaging via session tools
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },

    // Sandbox tool policy (applied inside sandboxed sessions)
    sandbox: {
      tools: {
        allow: [
          "exec", "process", "read", "write", "edit",
          "apply_patch",
          "sessions_list", "sessions_history", "sessions_send",
          "sessions_spawn", "session_status",
        ],
        deny: ["browser", "canvas", "nodes", "cron", "discord", "gateway"],
      },
    },
  },
}
```

---

## `models` — custom providers and base URLs

Add custom model providers or override built-in ones:

```json5
{
  models: {
    mode: "merge",                   // merge (default) | replace

    // Custom provider map
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions",   // openai-completions | openai-responses | anthropic-messages | google-generative-ai
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 32000,
          },
        ],
      },
    },

    // AWS Bedrock auto-discovery
    bedrockDiscovery: {
      enabled: false,
      region: "us-east-1",
      providerFilter: ["anthropic"],
      refreshInterval: "1h",
      defaultContextWindow: 200000,
      defaultMaxTokens: 8192,
    },
  },
}
```

**Provider field notes:**
- `api`: request adapter type. Use `openai-completions` for Ollama and LiteLLM proxies, `anthropic-messages` for Anthropic-compatible APIs.
- `auth`: `api-key` (default), `token`, `oauth`, `aws-sdk`.
- `authHeader: true`: force credential in the `Authorization` header.
- `headers`: extra static headers for proxy/tenant routing.
- `injectNumCtxForOpenAICompat`: for Ollama, inject `options.num_ctx` (default `true`).

See [Custom providers](/gateway/configuration-reference#custom-providers-and-base-urls) for complete provider examples (Cerebras, Z.AI, Moonshot, MiniMax, LM Studio, etc.).

---

## `skills` — skills configuration

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm",            // npm | pnpm | yarn
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: optional allowlist for bundled skills only. Managed/workspace skills are unaffected.
- `entries.<skillKey>.enabled: false`: disables a skill even if bundled or installed.

---

## `plugins` — plugin configuration

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-extension"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: { allowPromptInjection: false },
        config: { provider: "twilio" },
        env: { TWILIO_ACCOUNT_SID: "AC..." },
        subagent: {
          allowModelOverride: true,
          allowedModels: ["provider/model", "*"],
        },
      },
    },
    slots: {
      memory: "none",                // active memory plugin id, or "none" to disable
      contextEngine: "legacy",       // active context engine plugin id
    },
  },
}
```

- Loaded from `~/.openclaw/extensions`, `<workspace>/.openclaw/extensions`, and `plugins.load.paths`.
- `allow`: optional allowlist; only listed plugins load. `deny` wins over `allow`.
- `plugins.entries.<id>.apiKey`: plugin-level API key convenience field.
- `plugins.entries.<id>.env`: plugin-scoped env var map.
- `plugins.entries.<id>.subagent.allowModelOverride`: allow plugin to request per-run provider/model overrides for background subagent runs.
- `plugins.entries.<id>.subagent.allowedModels`: optional allowlist of `provider/model` targets for trusted subagent overrides.
- `plugins.entries.<id>.hooks.allowPromptInjection`: when `false`, core blocks prompt-mutating hook fields.
- `plugins.installs`: CLI-managed install metadata used by `openclaw plugins update`. Treat as managed state; prefer CLI commands over manual edits.
- **Config changes require a gateway restart.**

See [Plugins](/tools/plugin).

---

## `browser` — browser automation

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: true,  // default: true (trusted-network mode)
      // Set false for strict public-only navigation
      // hostnameAllowlist: ["*.example.com"],
    },
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: { cdpPort: 18801, color: "#0066CC" },
      user: { driver: "existing-session", attachOnly: true, color: "#00AA00" },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222" },
    },
    color: "#FF4500",
    // headless: false,
    // noSandbox: false,
    // extraArgs: [],
    // executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
  },
}
```

- `ssrfPolicy.dangerouslyAllowPrivateNetwork: false`: strict public-only mode. Use `hostnameAllowlist` for exceptions.
- `existing-session` profiles use Chrome MCP instead of CDP and can target any Chromium-based browser.
- Auto-detect order: default browser → Chrome → Brave → Edge → Chromium → Chrome Canary.

---

## `ui` — Control UI customization

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB",                  // emoji, short text, image URL, or data URI
    },
  },
}
```

- `seamColor`: accent color for native app UI chrome (Talk Mode bubble tint, etc.).
- `assistant`: Control UI identity override. Falls back to active agent identity.

---

## `gateway` — gateway networking and auth

```json5
{
  gateway: {
    mode: "local",                   // local | remote
    port: 18789,                     // default; precedence: --port > env > config > 18789
    bind: "loopback",                // auto | loopback | lan | tailnet | custom

    auth: {
      mode: "token",                 // none | token | password | trusted-proxy
      token: "your-token",           // or OPENCLAW_GATEWAY_TOKEN env
      // password: "your-password",  // or OPENCLAW_GATEWAY_PASSWORD env
      // trustedProxy: { userHeader: "x-forwarded-user" },
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,
      },
    },

    tailscale: {
      mode: "off",                   // off | serve | funnel
      resetOnExit: false,
    },

    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      // root: "dist/control-ui",
      // allowedOrigins: ["https://control.example.com"],
      // dangerouslyAllowHostHeaderOriginFallback: false,
      // allowInsecureAuth: false,
      // dangerouslyDisableDeviceAuth: false,
    },

    remote: {
      url: "ws://gateway.tailnet:18789",
      transport: "ssh",              // ssh | direct
      token: "your-token",
      // password: "your-password",
    },

    trustedProxies: ["10.0.0.1"],
    allowRealIpFallback: false,

    tools: {
      deny: ["browser"],
      allow: ["gateway"],
    },

    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000,
        },
      },
    },

    channelHealthCheckMinutes: 5,
    channelStaleEventThresholdMinutes: 30,
    channelMaxRestartsPerHour: 10,

    http: {
      endpoints: {
        chatCompletions: { enabled: false },
        responses: {
          enabled: false,
          // maxUrlParts: 10,
          // files: { urlAllowlist: [], allowUrl: true },
          // images: { urlAllowlist: [], allowUrl: true },
        },
      },
      securityHeaders: {
        // strictTransportSecurity: "...",  // for HTTPS origins only
      },
    },
  },
}
```

**Bind modes:**
- `loopback` (default): `127.0.0.1` only. Safe for local use.
- `lan`: `0.0.0.0` (all interfaces). Required in Docker with bridge networking.
- `tailnet`: Tailscale IP only.
- `auto`: detect best bind address.
- `custom`: use `customBindHost`.

**Auth modes:**
- `token`: shared bearer token (recommended).
- `password`: basic password.
- `trusted-proxy`: delegate auth to an identity-aware reverse proxy.
- `none`: no authentication. Only for trusted local loopback.

**Multi-instance isolation:** Use `--dev` flag (port 19001, `~/.openclaw-dev`) or `--profile <name>` for separate gateways.

See [Gateway](/gateway/index), [Authentication](/gateway/authentication), and [Trusted Proxy Auth](/gateway/trusted-proxy-auth).

---

## `hooks` — webhook ingress

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    maxBodyBytes: 262144,
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: false,
    allowedSessionKeyPrefixes: ["hook:"],
    allowedAgentIds: ["hooks", "main"],
    presets: ["gmail"],
    transformsDir: "~/.openclaw/hooks/transforms",
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        agentId: "hooks",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "From: {{messages[0].from}}\nSubject: {{messages[0].subject}}\n{{messages[0].snippet}}",
        deliver: true,
        channel: "last",
        model: "openai/gpt-5.2-mini",
      },
    ],

    // Gmail watch integration
    gmail: {
      account: "openclaw@gmail.com",
      topic: "projects/<project-id>/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,
      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },
      model: "openrouter/meta-llama/llama-3.3-70b-instruct:free",
      thinking: "off",
    },
  },
}
```

**Endpoints:**
- `POST /hooks/wake` — `{ text, mode?: "now"|"next-heartbeat" }`
- `POST /hooks/agent` — `{ message, name?, agentId?, sessionKey?, wakeMode?, deliver?, channel?, model? }`
- `POST /hooks/<name>` — resolved via `hooks.mappings`

Auth: `Authorization: Bearer <token>` or `x-openclaw-token: <token>`.

---

## `canvasHost` — agent-editable canvas

```json5
{
  canvasHost: {
    root: "~/.openclaw/workspace/canvas",
    liveReload: true,
    // enabled: false,  // or OPENCLAW_SKIP_CANVAS_HOST=1
  },
}
```

- Serves HTML/CSS/JS at `http://<gateway-host>:<port>/__openclaw__/canvas/`.
- Also serves A2UI at `http://<gateway-host>:<port>/__openclaw__/a2ui/`.
- **Changes require a gateway restart.**

---

## `discovery` — mDNS and DNS-SD

```json5
{
  discovery: {
    // mDNS (Bonjour/Zeroconf)
    mdns: {
      mode: "minimal",              // minimal | full | off
      // Override hostname: OPENCLAW_MDNS_HOSTNAME env
    },

    // Wide-area DNS-SD
    wideArea: { enabled: true },
  },
}
```

- `minimal` (default): omits `cliPath` and `sshPort` from TXT records.
- `full`: includes `cliPath` and `sshPort`.
- **Discovery changes require a gateway restart.**

---

## `secrets` — secret provider configuration

Configure how SecretRefs are resolved. The `SecretRef` shape is:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

```json5
{
  secrets: {
    providers: {
      default: { source: "env" },

      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",                  // json | singleValue
        timeoutMs: 5000,
      },

      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        passEnv: ["PATH", "VAULT_ADDR"],
        // allowSymlinkCommand: false,
        // trustedDirs: ["/usr/local/bin"],
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

**SecretRef validation:**
- `source: "env"` id: `^[A-Z][A-Z0-9_]{0,127}$`
- `source: "file"` id: absolute JSON pointer (for example `/providers/openai/apiKey`)
- `source: "exec"` id: `^[A-Za-z0-9][A-Za-z0-9._:/-]{0,255}$`

Secrets are resolved eagerly at activation time. Unresolved refs on enabled surfaces block startup/reload. Inactive surfaces emit diagnostics and do not block.

See [Secrets Management](/gateway/secrets) and [SecretRef Credential Surface](/reference/secretref-credential-surface).

---

## `cron` — cron job defaults

```json5
{
  cron: {
    enabled: true,
    maxConcurrentRuns: 2,
    sessionRetention: "24h",         // duration string or false
    runLog: {
      maxBytes: "2mb",
      keepLines: 2000,
    },
    webhook: "https://example.invalid/legacy", // deprecated fallback
    webhookToken: "replace-with-dedicated-token",
  },
}
```

Cron jobs themselves are stored in the cron store, not in `openclaw.json`. These are global defaults and limits.

See [Cron Jobs](/automation/cron-jobs).

---

## `wizard` — guided-setup metadata

Written automatically by `openclaw onboard`, `openclaw configure`, and `openclaw doctor`. Do not edit by hand.

```json5
{
  wizard: {
    lastRunAt: "2026-01-01T00:00:00.000Z",
    lastRunVersion: "2026.1.4",
    lastRunCommit: "abc1234",
    lastRunCommand: "configure",
    lastRunMode: "local",
  },
}
```

---

## `$include` — config file splitting

Split large configs across multiple files:

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: {
    $include: ["./clients/mueller.json5", "./clients/schmidt.json5"],
  },
}
```

**Merge behavior:**
- Single file: replaces the containing object.
- Array of files: deep-merged in order (later overrides earlier).
- Sibling keys: merged after includes (override included values).
- Nested includes: up to 10 levels deep.
- Paths: resolved relative to the including file, must stay inside the top-level config directory (`dirname` of `openclaw.json`). Absolute/`../` forms are allowed only when they still resolve inside that boundary.
- Errors: clear messages for missing files, parse errors, and circular includes.

---

## `$schema` — editor schema attachment

The only root-level key that is not validated strictly:

```json5
{
  "$schema": "https://schema.openclaw.ai/openclaw.json",
}
```

Use this to enable JSON Schema validation and autocomplete in editors like VS Code.

---

## Hot reload

The gateway watches `openclaw.json` for changes. Most changes apply without downtime (hybrid mode, the default):

| Applies without restart | Requires restart |
| ----------------------- | ---------------- |
| channels                | gateway (port, bind, auth, TLS) |
| agents, models          | discovery (mDNS/DNS-SD) |
| hooks, cron, skills     | canvasHost |
| session, messages, tools| plugins |
| media, browser          | |
| UI, auth profiles       | |

Configure reload behavior with `gateway.hotReload`:

```json5
{
  gateway: {
    hotReload: "hybrid", // hybrid | hot | restart | off
  },
}
```

---

## Media template variables

Template placeholders expanded in `tools.media.models[].args`:

| Variable           | Description                                |
| ------------------ | ------------------------------------------ |
| `{{Body}}`         | Full inbound message body                  |
| `{{RawBody}}`      | Raw body (no history/sender wrappers)      |
| `{{BodyStripped}}` | Body with group mentions stripped          |
| `{{From}}`         | Sender identifier                          |
| `{{To}}`           | Destination identifier                     |
| `{{MessageSid}}`   | Channel message id                         |
| `{{SessionId}}`    | Current session UUID                       |
| `{{IsNewSession}}` | `"true"` when new session created          |
| `{{MediaUrl}}`     | Inbound media pseudo-URL                   |
| `{{MediaPath}}`    | Local media path                           |
| `{{MediaType}}`    | Media type (image/audio/document/…)        |
| `{{Transcript}}`   | Audio transcript                           |
| `{{Prompt}}`       | Resolved media prompt for CLI entries      |
| `{{MaxChars}}`     | Resolved max output chars for CLI entries  |
| `{{ChatType}}`     | `"direct"` or `"group"`                    |
| `{{GroupSubject}}` | Group subject (best effort)                |
| `{{GroupMembers}}` | Group members preview (best effort)        |
| `{{SenderName}}`   | Sender display name (best effort)          |
| `{{SenderE164}}`   | Sender phone number (best effort)          |
| `{{Provider}}`     | Channel provider hint (whatsapp, telegram…)|

---

## Related

- [Configuration](/gateway/configuration) — task-oriented setup guide
- [Configuration Reference](/gateway/configuration-reference) — exhaustive field-by-field reference
- [Configuration Examples](/gateway/configuration-examples) — complete copy-paste examples
- [Secrets Management](/gateway/secrets) — credential management and SecretRef
- [Gateway Runbook](/gateway/index) — startup and operations
- [Doctor](/gateway/doctor) — diagnose and repair config issues
