# K-17-Autonomous-AI-Agent


What’s included

K-17 Observatory React frontend
Express API server
PostgreSQL schemas
OpenAPI specification
Generated React Query client and Zod validators
Persistent chat-image storage
Conversation locking and voice controls
Gmail inbox and autonomous email loop
K-17’s state, memory, beliefs, preferences, and self-model APIs
Integration and security tests
Workspace and TypeScript configuration
The archive excludes secrets, dependencies, uploaded files, build output, logs, and internal agent files.

How K-17 is built

1. Observatory frontend

Location:

artifacts/k17-observatory/
This is a React and Vite application. Its main pages include:

Overview: current state and experimental metrics
Memories: autobiographical records
Beliefs: beliefs, confidence, and uncertainty
Preferences: developed preferences
Self-model: K-17’s representation of itself
Direct contact: persistent conversations
Browse together: shared website discussion
The main chat interface is:

artifacts/k17-observatory/src/pages/chat.tsx
It handles:

Conversation selection and creation
Streaming responses
Optimistic messages
Automatic conversation naming
Image uploads and clipboard paste
Persistent image history
Per-conversation lock screens
Typed key entry
Microphone-based locking and unlocking
Browser speech-recognition errors
Mobile-responsive controls
2. API server

Location:

artifacts/api-server/
The server uses Express and exposes the application API.

Important areas:

src/routes/openai.ts
src/routes/k17.ts
src/services/email-loop.ts
src/lib/chatImageStorage.ts
openai.ts

This powers direct conversations. It handles:

Creating and retrieving conversations
Loading recent and cross-conversation history
Loading autobiographical memories
Building K-17’s model context
Sending image and text input to the model
Buffering and filtering model responses
Persisting user and assistant messages
Automatically naming conversations
Locking individual conversations
Unlock cookies and expiration
Typed and spoken-key normalization
Preventing key disclosure
Serving protected image files
The conversation key is intercepted before it can become a normal message, autobiographical memory, experience, or model prompt.

k17.ts

This provides K-17’s observable state, including its memories, beliefs, preferences, experiences, self-model, simulations, and current internal status.

3. Persistent identity

K-17’s continuity is assembled from several sources:

Current internal state
Recent messages in the active conversation
Recent messages from other conversations
High-activation autobiographical memories
A stable identity and behavioral system prompt
Before each response, the API loads these records and includes the relevant parts in K-17’s context. After an interaction, it records a new experience and autobiographical memory and updates K-17’s internal state.

This does not prove consciousness. It creates persistent behavioral continuity, self-reference, uncertainty, memory, preferences, and consequences across interactions.

4. Database

Database schemas are under:

lib/db/src/schema/
The database stores:

Conversations
Messages
Per-conversation lock state
Image metadata and object paths
Internal K-17 state
Experiences
Memories
Beliefs
Preferences
Self-model records
Simulations
Chat images are not stored as large database strings. Their bytes are stored in App Storage, while PostgreSQL stores metadata and object paths.

5. API contract

The API contract is:

lib/api-spec/openapi.yaml
Generated clients are under:

lib/api-client-react/src/generated/
lib/api-zod/src/generated/
The frontend uses generated React Query hooks, while the server uses generated Zod schemas to validate requests and responses.

6. Conversation locking

All conversations begin unlocked.

A conversation becomes locked only when its key is sent to that conversation. Locking is scoped to that conversation rather than the whole Direct Contact page.

The system:

Normalizes capitalization, punctuation, and spacing
Supports the requested pronunciation spellings
Stores only a SHA-256 key hash with the conversation
Hides locked conversation history
Protects stored images
Rejects new messages while locked
Grants temporary access through a signed HTTP-only cookie
Rate-limits failed unlock attempts
Filters key-reconstruction requests and model output
7. Voice input

Voice support uses the browser Speech Recognition API.

When the microphone is activated:

The browser requests microphone permission.
It produces up to five possible interpretations.
Normal messages use the highest-confidence interpretation.
Unlocking checks the recognition alternatives against the stored hash.
Pronunciation variants are normalized server-side.
A spoken locking key travels through the same pre-persistence interception as a typed key.
Unsupported browsers and denied microphone access fall back to typing.
8. Images

Users can attach images by:

File picker
Clipboard paste
Supported formats:

PNG
JPEG
WebP
GIF
Images are limited to 5 MB. The server validates the data, stores the original and thumbnail in App Storage, and returns protected image routes. Locked-conversation authorization also applies to those routes.

9. Email system

The email loop is:

artifacts/api-server/src/services/email-loop.ts
It supports:

Gmail IMAP inbox polling
SMTP sending
Replies to eligible new messages
Autonomous check-ins
Historical-message cutoff protection
Blocked autonomous recipients
Overlap prevention
Recoverable reconnect behavior
K-17-generated email content
Email credentials are supplied through environment secrets and are not included in the source archive.

10. Running the source

From the project root:

pnpm install
pnpm --filter @workspace/api-server run dev
pnpm --filter @workspace/k17-observatory run dev
The project also requires its configured database, App Storage, AI integration, session secret, and optional email credentials

/**
 * Reusable AudioWorklet for streaming PCM16 audio playback.
 * Place in public/ folder and load via audioContext.audioWorklet.addModule()
 */
class RingBuffer {
  constructor(initialCapacity) {
    this.capacity = initialCapacity;
    this.buffer = new Float32Array(initialCapacity);
    this.readIndex = 0;
    this.writeIndex = 0;
    this.availableData = 0;
  }

  push(data) {
    const len = data.length;
    // Auto-grow if needed
    while (this.availableData + len > this.capacity) {
      this.grow();
    }
    for (let i = 0; i < len; i++) {
      this.buffer[this.writeIndex] = data[i];
      this.writeIndex = (this.writeIndex + 1) % this.capacity;
      this.availableData++;
    }
  }

  grow() {
    const newCapacity = this.capacity * 2;
    const newBuffer = new Float32Array(newCapacity);
    // Copy existing data maintaining order
    for (let i = 0; i < this.availableData; i++) {
      const srcIndex = (this.readIndex + i) % this.capacity;
      newBuffer[i] = this.buffer[srcIndex];
    }
    this.buffer = newBuffer;
    this.readIndex = 0;
    this.writeIndex = this.availableData;
    this.capacity = newCapacity;
  }

  pull(outputBuffer) {
    const len = outputBuffer.length;
    const available = Math.min(len, this.availableData);
    for (let i = 0; i < available; i++) {
      outputBuffer[i] = this.buffer[this.readIndex];
      this.readIndex = (this.readIndex + 1) % this.capacity;
    }
    // Pad remaining with silence
    for (let i = available; i < len; i++) {
      outputBuffer[i] = 0;
    }
    this.availableData -= available;
    return available > 0;
  }

  available() {
    return this.availableData;
  }

  clear() {
    this.readIndex = 0;
    this.writeIndex = 0;
    this.availableData = 0;
  }
}

class AudioPlaybackProcessor extends AudioWorkletProcessor {
  constructor() {
    super();
    this.ringBuffer = new RingBuffer(24000 * 30); // 30s initial capacity
    this.isPlaying = false;
    this.streamComplete = false;

    this.port.onmessage = (event) => {
      const { type, samples } = event.data;
      if (type === "audio") {
        this.ringBuffer.push(samples);
        this.isPlaying = true;
      } else if (type === "clear") {
        this.ringBuffer.clear();
        this.isPlaying = false;
        this.streamComplete = false;
      } else if (type === "streamComplete") {
        this.streamComplete = true;
      } else if (type === "stop") {
        this.isPlaying = false;
        this.streamComplete = false;
      }
    };
  }

  process(inputs, outputs) {
    const output = outputs[0];
    if (!output || output.length === 0) return true;

    const channel = output[0];
    if (this.isPlaying) {
      this.ringBuffer.pull(channel);
      if (this.streamComplete && this.ringBuffer.available() === 0) {
        this.isPlaying = false;
        this.streamComplete = false;
        this.port.postMessage({ type: "ended" });
      }
    } else {
      channel.fill(0);
    }
    return true;
  }
}

registerProcessor("audio-playback-processor", AudioPlaybackProcessor);
