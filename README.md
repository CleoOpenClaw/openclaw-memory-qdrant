# openclaw-memory-qdrant (Cleo fork)

> Forked from [zuiho-kai/openclaw-memory-qdrant](https://github.com/zuiho-kai/openclaw-memory-qdrant). English translation and updated default storage path.

Local semantic memory plugin for OpenClaw, powered by Qdrant and Transformers.js. Zero-config semantic search, fully local, no API keys required.

**ClawHub**: https://clawhub.ai/skills/memory-qdrant

## Features

- **Local semantic search** - Generates embeddings locally with Transformers.js
- **In-memory mode** - Zero config, no external services needed
- **Auto-capture** - Automatically records important info via lifecycle hooks
- **Smart recall** - Retrieves relevant memories based on context

## Installation

### Via ClawHub (recommended)

```bash
clawhub install memory-qdrant
```

### Manual installation

```bash
cd ~/.openclaw/plugins
git clone https://github.com/CleoOpenClaw/openclaw-memory-qdrant.git memory-qdrant
cd memory-qdrant
npm install
```

### Requirements

**First-run preparation:**

1. **Node.js version**: Requires Node.js >= 18.17
   ```bash
   node --version  # Check version
   ```

2. **Build tools** (for compiling native dependencies):
   - **Windows**: Visual Studio Build Tools
     ```powershell
     npm install --global windows-build-tools
     ```
   - **macOS**: Xcode Command Line Tools
     ```bash
     xcode-select --install
     ```
   - **Linux**: build-essential
     ```bash
     sudo apt-get install build-essential  # Debian/Ubuntu
     sudo yum groupinstall "Development Tools"  # RHEL/CentOS
     ```

3. **Network access**:
   - Installation requires access to npmjs.com for dependencies
   - First run downloads an embedding model from huggingface.co (~25MB)
   - If using an external Qdrant server, network access to that server is required

4. **Native dependencies**:
   - `sharp`: Image processing library (may require compilation)
   - `onnxruntime`: ML inference engine (may require compilation)
   - `undici`: HTTP client (via @qdrant/js-client-rest)

### Recommended installation

```bash
# Use npm ci for reproducible installs (recommended for production)
npm ci

# Or install step by step (for debugging)
npm install --ignore-scripts  # Skip post-install scripts
npm rebuild                    # Then rebuild native modules
```

### Troubleshooting

**Problem: Native module compilation fails**
- Ensure build tools are installed for your platform
- Try clearing the cache: `npm cache clean --force`
- Delete node_modules and reinstall: `rm -rf node_modules && npm install`

**Problem: Model download fails**
- Check network connection and firewall settings
- Ensure huggingface.co is accessible
- The model is cached in `~/.cache/huggingface/`

**Problem: Node version incompatible**
- Upgrade to Node.js 18.17 or higher
- Use nvm to manage multiple Node versions: `nvm install 18 && nvm use 18`

## Configuration

Enable the plugin in your OpenClaw config:

```json
{
  "plugins": {
    "memory-qdrant": {
      "enabled": true,
      "autoCapture": false,
      "autoRecall": true,
      "captureMaxChars": 500
    }
  }
}
```

### Options

- **qdrantUrl** (optional): External Qdrant server URL. Leave empty for in-memory mode.
- **persistToDisk** (default: true): Save memories to disk in memory mode.
  - Data stored in `~/.openclaw/workspace/memory/semantic/` (or custom path)
  - Data survives restarts
  - Set to false for volatile in-memory mode (cleared on restart)
  - Only applies in memory mode (when qdrantUrl is not set)
- **storagePath** (optional): Custom storage directory.
  - Leave empty for default `~/.openclaw/workspace/memory/semantic/`
  - Supports `~` for home directory
  - Only applies when `persistToDisk: true`
- **autoCapture** (default: false): Automatically record conversation content.
  - **Privacy protection**: By default, text containing PII (emails, phone numbers) is skipped
  - Requires `allowPIICapture` to capture PII
- **allowPIICapture** (default: false): Allow capturing text containing PII.
  - **Privacy risk**: Only enable if you understand the implications
  - Requires `autoCapture` to be enabled
- **autoRecall** (default: true): Auto-inject relevant memories into conversations.
- **captureMaxChars** (default: 500): Maximum characters per captured memory.
- **maxMemorySize** (default: 1000): Maximum number of memories in in-memory mode.
  - Only applies in memory mode (when qdrantUrl is not set)
  - Oldest memories are auto-deleted when limit is reached (LRU eviction)
  - Range: 100-1000000
  - Set to 999999 for unlimited (no auto-deletion)
  - Unlimited mode may cause high memory usage; use with caution
  - External Qdrant mode is not subject to this limit

## Privacy & Security

### Data Storage

- **Disk persistence** (default): Data saved to `~/.openclaw/workspace/memory/semantic/` and restored on restart.
  - Set `persistToDisk: false` for volatile in-memory mode (cleared on restart)
- **Qdrant mode**: If `qdrantUrl` is configured, data is sent to that server.
  - Only configure trusted Qdrant servers
  - Use a local Qdrant instance or dedicated service account

### Network Access

- **First run**: Transformers.js downloads a model file from Hugging Face (~25MB)
- **Runtime**: Memory mode requires no network; Qdrant mode connects to the configured server

### Auto-capture

- **autoCapture** is disabled by default
- **PII protection**: Even with autoCapture enabled, text containing emails or phone numbers is skipped by default
- **allowPIICapture**: Set to true to capture PII text
  - Only enable if you understand the privacy risks
  - Suitable for: personal notes, test environments
  - Not suitable for: shared environments, production, handling others' data
- Recommended for personal use only; avoid enabling in shared or production environments

### Recommendations

1. Test in an isolated environment first
2. Review `index.js` to understand data handling logic
3. Lock dependency versions in sensitive environments (`npm ci`)
4. Periodically review stored memories

## Usage

The plugin provides three tools:

### memory_store
Save important information to long-term memory:

```javascript
memory_store({
  text: "User prefers Opus for complex tasks",
  category: "preference",
  importance: 0.8
})
```

### memory_search
Search for relevant memories:

```javascript
memory_search({
  query: "workflow preferences",
  limit: 5
})
```

### memory_forget
Delete a specific memory:

```javascript
memory_forget({
  memoryId: "uuid-here"
})
// Or search and delete
memory_forget({
  query: "content to delete"
})
```

## CLI Commands

The plugin registers CLI commands under `memory-qdrant` for managing memories outside agent sessions.

### Search memories

```bash
openclaw memory-qdrant search "workflow preferences"
```

### Store a single memory

```bash
openclaw memory-qdrant store "User prefers TypeScript over JavaScript"
openclaw memory-qdrant store "Deploy target is AWS eu-west-1" --category decision --importance 0.9
```

Options:
- `--category <category>` — one of: fact, preference, decision, entity, other (default: fact)
- `--importance <number>` — 0.0 to 1.0 (default: 0.8)

### Bulk import memories

```bash
# From a JSON file (array of { text, category?, importance? })
openclaw memory-qdrant import seeds.json

# From a text file (one memory per line, category defaults to "fact")
openclaw memory-qdrant import notes.txt

# Preview without storing
openclaw memory-qdrant import seeds.json --dry-run
```

### Clear all memories

```bash
openclaw memory-qdrant clear --confirm
```

Requires `--confirm` flag to prevent accidental deletion.

### View statistics

```bash
openclaw memory-qdrant stats
```

Shows total count, storage mode, persist path, and category breakdown.

## Technical Details

### Architecture

- **Vector DB**: Qdrant (in-memory or external)
- **Embedding model**: Xenova/all-MiniLM-L6-v2 (runs locally)
- **Module system**: ES6 modules

### Key Implementation

The plugin uses a **factory function pattern** to export tools, ensuring compatibility with OpenClaw's tool system:

```javascript
export default {
  name: 'memory-qdrant',
  version: '1.0.0',
  tools: [
    () => ({
      name: 'memory_search',
      description: '...',
      parameters: { ... },
      execute: async (params) => { ... }
    })
  ]
}
```

### FAQ

**Q: Why use factory functions?**

A: OpenClaw's tool system calls `tool.execute()`. Exporting objects directly causes `tool.execute is not a function` errors. Factory functions ensure a new tool instance is returned on each call.

**Q: Why ES6 modules?**

A: OpenClaw's plugin loader expects ES6 module format. Set `"type": "module"` in `package.json`.

**Q: Where is data stored?**

A: In memory mode with `persistToDisk: true` (default), data is saved to `~/.openclaw/workspace/memory/semantic/` and survives restarts. With `persistToDisk: false`, data is only kept in memory and cleared on restart.

## Development

```bash
# Install dependencies
npm install

# Test (requires OpenClaw environment)
openclaw gateway restart
```

## License

MIT

## Credits

- Original: [zuiho-kai/openclaw-memory-qdrant](https://github.com/zuiho-kai/openclaw-memory-qdrant)
- [Qdrant](https://qdrant.tech/) - Vector database
- [Transformers.js](https://huggingface.co/docs/transformers.js) - Local ML inference
- [OpenClaw](https://openclaw.ai/) - AI assistant framework
