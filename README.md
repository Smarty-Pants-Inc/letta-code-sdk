# Letta Code SDK

[![npm](https://img.shields.io/npm/v/@letta-ai/letta-code-sdk.svg?style=flat-square)](https://www.npmjs.com/package/@letta-ai/letta-code-sdk) [![Discord](https://img.shields.io/badge/discord-join-blue?style=flat-square&logo=discord)](https://discord.gg/letta)

The SDK interface to [Letta Code](https://github.com/letta-ai/letta-code). Build agents with persistent memory that learn over time.

```typescript
import { prompt } from '@letta-ai/letta-code-sdk';

const result = await prompt('Find and fix the bug in auth.py', {
  allowedTools: ['Read', 'Edit', 'Bash'],
  permissionMode: 'bypassPermissions'
});
console.log(result.result);
```

## Installation

```bash
npm install @letta-ai/letta-code-sdk
```

## Quick Start

### One-shot prompt

```typescript
import { prompt } from '@letta-ai/letta-code-sdk';

const result = await prompt('Run: echo hello', {
  allowedTools: ['Bash'],
  permissionMode: 'bypassPermissions'
});
console.log(result.result); // "hello"
```

### Multi-turn session

```typescript
import { createSession } from '@letta-ai/letta-code-sdk';

await using session = createSession();

await session.send('What is 5 + 3?');
for await (const msg of session.stream()) {
  if (msg.type === 'assistant') console.log(msg.content);
}

await session.send('Multiply that by 2');
for await (const msg of session.stream()) {
  if (msg.type === 'assistant') console.log(msg.content);
}
```

### Persistent memory

Agents persist across sessions and remember context:

```typescript
import { createSession, resumeSession } from '@letta-ai/letta-code-sdk';

// First session
const session1 = createSession();
await session1.send('Remember: the secret word is "banana"');
for await (const msg of session1.stream()) { /* ... */ }
const agentId = session1.agentId;
session1.close();

// Later...
await using session2 = resumeSession(agentId);
await session2.send('What is the secret word?');
for await (const msg of session2.stream()) {
  if (msg.type === 'assistant') console.log(msg.content); // "banana"
}
```

### Multi-threaded Conversations

Run multiple concurrent conversations with the same agent. Each conversation has its own message history while sharing the agent's persistent memory.

```typescript
import { createSession, resumeSession, resumeConversation } from '@letta-ai/letta-code-sdk';

// Create an agent
const session = createSession();
await session.send('Hello!');
for await (const msg of session.stream()) { /* ... */ }
const agentId = session.agentId;
const conversationId = session.conversationId; // Save this!
session.close();

// Resume a specific conversation
await using session2 = resumeConversation(conversationId);
await session2.send('Continue our discussion...');
for await (const msg of session2.stream()) { /* ... */ }

// Create a NEW conversation on the same agent
await using session3 = resumeSession(agentId, { newConversation: true });
await session3.send('Start a fresh thread...');
// session3.conversationId is different from conversationId

// Resume with agent's default conversation
await using session4 = resumeSession(agentId, { defaultConversation: true });

// Resume last used session (agent + conversation)
await using session5 = createSession({ continue: true });

// Create new agent with a new (non-default) conversation
await using session6 = createSession({ newConversation: true });
```

**Key concepts:**
- **Agent** (`agentId`): Persistent entity with memory that survives across sessions
- **Conversation** (`conversationId`): A message thread within an agent
- **Session** (`sessionId`): A single execution/connection

Agents remember across conversations (via memory blocks), but each conversation has its own message history.

## Agent Configuration

### System Prompt

Choose from built-in presets or provide a custom prompt:

```typescript
// Use a preset
createSession({
  systemPrompt: { type: 'preset', preset: 'letta-claude' }
});

// Use a preset with additional instructions
createSession({
  systemPrompt: { 
    type: 'preset', 
    preset: 'letta-claude',
    append: 'Always respond in Spanish.'
  }
});

// Use a completely custom prompt
createSession({
  systemPrompt: 'You are a helpful Python expert.'
});
```

**Available presets:**
- `default` / `letta-claude` - Full Letta Code prompt (Claude-optimized)
- `letta-codex` - Full Letta Code prompt (Codex-optimized)
- `letta-gemini` - Full Letta Code prompt (Gemini-optimized)
- `claude` - Basic Claude (no skills/memory instructions)
- `codex` - Basic Codex
- `gemini` - Basic Gemini

### Memory Blocks

Configure which memory blocks the agent uses:

```typescript
// Use default blocks (persona, human, project)
createSession({});

// Use specific preset blocks
createSession({
  memory: ['project', 'persona']  // Only these blocks
});

// Use custom blocks
createSession({
  memory: [
    { label: 'context', value: 'API documentation for Acme Corp...' },
    { label: 'rules', value: 'Always use TypeScript. Prefer functional patterns.' }
  ]
});

// Mix presets and custom blocks
createSession({
  memory: [
    'project',  // Use default project block
    { label: 'custom', value: 'Additional context...' }
  ]
});

// No optional blocks (only core skills blocks)
createSession({
  memory: []
});
```

### Convenience Props

Quickly customize common memory blocks:

```typescript
createSession({
  persona: 'You are a senior Python developer who writes clean, tested code.',
  human: 'Name: Alice. Prefers concise responses.',
  project: 'FastAPI backend for a todo app using PostgreSQL.'
});

// Combine with memory config
createSession({
  memory: ['persona', 'project'],  // Only include these blocks
  persona: 'You are a Go expert.',
  project: 'CLI tool for managing Docker containers.'
});
```

### Tool Execution

Execute tools with automatic permission handling:

```typescript
import { prompt } from '@letta-ai/letta-code-sdk';

// Run shell commands
const result = await prompt('List all TypeScript files', {
  allowedTools: ['Glob', 'Bash'],
  permissionMode: 'bypassPermissions',
  cwd: '/path/to/project'
});

// Read and analyze code
const analysis = await prompt('Explain what auth.ts does', {
  allowedTools: ['Read', 'Grep'],
  permissionMode: 'bypassPermissions'
});
```

## API Reference

### Functions

| Function | Description |
|----------|-------------|
| `prompt(message, options?)` | One-shot query, returns result directly |
| `createSession(options?)` | Create new agent session |
| `resumeSession(agentId, options?)` | Resume existing agent by ID |
| `resumeConversation(conversationId, options?)` | Resume specific conversation (derives agent automatically) |

### Session

| Property/Method | Description |
|-----------------|-------------|
| `send(message)` | Send user message |
| `stream()` | AsyncGenerator yielding messages |
| `close()` | Close the session |
| `agentId` | Agent ID (for resuming later) |
| `sessionId` | Current session ID |
| `conversationId` | Conversation ID (for resuming specific thread) |

### Options

```typescript
interface SessionOptions {
  // Model selection
  model?: string;

  // Conversation options
  conversationId?: string;      // Resume specific conversation
  newConversation?: boolean;    // Create new conversation on agent
  continue?: boolean;           // Resume last session (agent + conversation)
  defaultConversation?: boolean; // Use agent's default conversation

  // System prompt: string or preset config
  systemPrompt?: string | {
    type: 'preset';
    preset: 'default' | 'letta-claude' | 'letta-codex' | 'letta-gemini' | 'claude' | 'codex' | 'gemini';
    append?: string;
  };

  // Memory blocks: preset names, custom blocks, or mixed
  memory?: Array<string | CreateBlock | { blockId: string }>;

  // Convenience: set block values directly
  persona?: string;
  human?: string;
  project?: string;

  // Tool configuration
  allowedTools?: string[];
  permissionMode?: 'default' | 'acceptEdits' | 'bypassPermissions';

  // Working directory
  cwd?: string;
}
```

### Message Types

```typescript
// Streamed during receive()
interface SDKAssistantMessage {
  type: 'assistant';
  content: string;
  uuid: string;
}

// Final message
interface SDKResultMessage {
  type: 'result';
  success: boolean;
  result?: string;
  error?: string;
  durationMs: number;
  conversationId: string;
}
```

## Examples

See [`examples/`](./examples/) for comprehensive examples including:

- Basic session usage
- Multi-turn conversations
- Session resume with persistent memory
- **Multi-threaded conversations** (resumeConversation, newConversation)
- System prompt configuration
- Memory block customization
- Tool execution (Bash, Glob, Read, etc.)

Run examples:
```bash
bun examples/v2-examples.ts all

# Run just conversation tests
bun examples/v2-examples.ts conversations
```

## License

Apache-2.0
