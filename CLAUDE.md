# CLAUDE.md

## Project Overview

This is a **Zoom Meeting Bot** application built with Recall.ai that demonstrates two key capabilities:
1. **Real-time transcript streaming** during active calls
2. **Post-call artifact retrieval** (MP4 video, MP3 audio, full transcript)

The bot joins Zoom meetings, streams live transcripts to the browser via WebSocket, and after the call ends, fetches and displays downloadable media files.

## Tech Stack

- **Framework**: Next.js 14 (API routes for backend)
- **Database**: PostgreSQL with Prisma ORM
- **Real-time**: WebSocket server (`ws` library) for live transcript broadcasting
- **External API**: Recall.ai Meeting Bot API
- **Tunneling**: ngrok (static domain required for webhooks)
- **Frontend**: React 18

## Architecture Overview

```
┌─────────────┐
│  Zoom Call  │
└──────┬──────┘
       │
       │ (bot joins)
       ▼
┌─────────────────┐
│  Recall.ai Bot  │
└────────┬────────┘
         │
         │ (webhooks)
         ▼
┌──────────────────────────────────┐
│  Next.js API Routes (ngrok URL)  │
│  - /api/webhook                  │
│  - /api/startRecall              │
│  - /api/userData                 │
└────────┬─────────────────────────┘
         │
         ├──► PostgreSQL (Prisma)
         │    - Meeting records
         │    - Transcript lines
         │
         └──► WebSocket Server (port 4000)
              - Broadcasts real-time transcripts
              - Browser connects and receives updates
```

## Key Files and Their Responsibilities

### API Routes (`pages/api/`)

#### `startRecall.ts`
**Purpose**: Creates a new Recall.ai bot and initiates recording

**Key functionality**:
- Generates unique `externalId` for tracking
- POSTs to `https://us-west-2.recall.ai/api/v1/bot` with:
  - `meeting_url`: Zoom link
  - `webhook_url`: Endpoint for bot status changes
  - `recording_config`: Configures video/audio/transcript settings
  - `realtime_endpoints`: Webhook URL for live transcript streaming
- Saves meeting record to PostgreSQL with `botId` and `externalId`
- **Important**: Uses raw API key (no "Bearer " prefix)

#### `webhook.ts`
**Purpose**: Receives webhooks from Recall.ai for both real-time and status events

**Handles two webhook types**:
1. **`transcript.data`** / **`transcript.partial_data`**: Real-time transcription
   - Saves transcript lines to DB
   - Broadcasts to WebSocket clients immediately
   - Fast ACK (responds quickly to avoid timeouts)

2. **`bot.status_change`**: Bot lifecycle events
   - `call_ended`: Call has finished, recording processing started
   - `done`: All artifacts ready, triggers media fetch
   - On `done`: Calls `waitForRecordingId()` → `fetchArtifacts()` → saves URLs to DB

#### `userData.ts`
**Purpose**: Polling endpoint for frontend to fetch meeting data

**Returns**:
- Latest meeting record
- Transcript lines (ordered by timestamp)
- `videoUrl` and `audioUrl` (when available post-call)

#### `manualRetrieve.ts`
**Purpose**: Manual trigger to fetch post-call artifacts

**Use case**: If automatic webhook handling fails or user wants to force refresh

### Library Files (`lib/`)

#### `recall-media.ts`
**Core media retrieval functions**:

1. **`waitForRecordingId(botId: string)`**
   - Polls `GET /bot/{botId}` until `recordings[0].id` appears
   - Retries with 3s delays (max 20 tries)
   - Returns `recordingId` or `null`

2. **`getMediaShortcuts(recordingId: string)`**
   - Fetches `GET /recording/{recordingId}/`
   - Returns `media_shortcuts` object with direct download URLs
   - Preferred method (faster, pre-signed URLs)

3. **`fetchArtifacts(recordingId: string)`**
   - First tries `media_shortcuts.video_mixed.data.download_url`
   - Falls back to `GET /video_mixed?recording_id={recordingId}`
   - Same pattern for audio (`/audio_mixed`)
   - Retries up to 10 times with 5s delays
   - Returns `{ videoUrl, audioUrl }`

4. **`fetchStructuredTranscriptByRecording(recordingId: string)`**
   - Fetches full structured transcript
   - Returns array of transcript objects with speaker info

#### `prisma.ts`
Standard Prisma client singleton pattern for Next.js.

### Database Schema (`prisma/schema.prisma`)

#### `Meeting` Model
```prisma
model Meeting {
  id          String      @id @default(cuid())
  userId      String
  meetingUrl  String
  externalId  String      @unique      // Our tracking ID
  botId       String                   // Recall.ai bot ID
  recordingId String                   // Populated after call ends
  videoUrl    String?                  // MP4 download URL
  audioUrl    String?                  // MP3 download URL
  transcript  Transcript[]
  createdAt   DateTime    @default(now())
}
```

#### `Transcript` Model
```prisma
model Transcript {
  id        String   @id @default(cuid())
  meetingId String
  meeting   Meeting  @relation(fields: [meetingId], references: [id])
  text      String
  speaker   String   @default("Unknown speaker")
  timestamp DateTime @default(now())
}
```

### WebSocket Server (`ws-server.ts`)

**Ports**: 4000 (default)

**Endpoints**:
- `ws://localhost:4000/recall` - WebSocket connection for browser clients
- POST `/send` - Internal endpoint to broadcast messages to all connected clients

**Usage**: Webhook handler POSTs new transcript data here, which broadcasts to all listening browsers.

## Data Flow Diagrams

### Real-Time Transcript Flow
```
Zoom Call (speech)
  → Recall.ai Bot (transcribes)
  → POST /api/webhook (transcript.data)
  → Save to PostgreSQL
  → POST to ws-server:4000/send
  → Broadcast via WebSocket
  → Browser receives and displays
```

### Post-Call Media Flow
```
Call ends
  → Recall.ai processes recording
  → POST /api/webhook (bot.status_change: done)
  → waitForRecordingId(botId)
  → GET /bot/{botId} (poll until recordings[] exists)
  → recordingId obtained
  → fetchArtifacts(recordingId)
  → media_shortcuts or video_mixed/audio_mixed APIs
  → videoUrl + audioUrl returned
  → Save to Meeting record
  → Frontend polls /api/userData
  → Display download links and players
```

## Recall.ai API Integration

### Base URL
`https://us-west-2.recall.ai/api/v1` (region may vary)

### Authentication
**Header format**: `Authorization: <RAW_API_KEY>`
- **NOT** `Bearer <token>`
- Raw key directly from `.env` `RECALL_API_KEY`

### Key Endpoints Used

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/bot` | POST | Create bot, join meeting |
| `/bot/{id}` | GET | Get bot status, check for `recordings[]` |
| `/recording/{id}` | GET | Get recording details, `media_shortcuts` |
| `/video_mixed` | GET | Fallback for video URL |
| `/audio_mixed` | GET | Fallback for audio URL |
| `/transcript` | GET | Get structured transcript by recording ID |

### Webhook Events

**Real-time events** (`realtime_endpoints`):
- `transcript.data`: Final transcript segment
- `transcript.partial_data`: In-progress (unstable) transcription

**Status events** (`webhook_url`):
- `bot.status_change`:
  - `in_waiting_room`: Bot waiting to be admitted
  - `in_call_not_recording`: Admitted but not recording yet
  - `in_call_recording`: Actively recording
  - `call_ended`: Call finished, processing started
  - `done`: All artifacts ready
  - `fatal`: Error occurred

## Development Workflow

### Initial Setup
1. Install dependencies: `pnpm install`
2. Configure `.env`:
   - `DATABASE_URL`: PostgreSQL connection string
   - `RECALL_API_KEY`: Raw API key (no "Bearer ")
3. Setup database: `npx prisma migrate dev`
4. Configure ngrok with static domain in `~/.config/ngrok/ngrok.yml`

### Running the App
**Three terminals required**:

1. **Terminal 1**: Next.js dev server
   ```bash
   npm run dev
   ```

2. **Terminal 2**: WebSocket relay
   ```bash
   npx ts-node ws-server.ts
   # or: npm run ws
   ```

3. **Terminal 3**: ngrok tunnel
   ```bash
   ngrok start --all
   ```

### Making Changes

#### Adding New Bot Configuration Options
1. Modify `startRecall.ts` → `recording_config` object
2. Reference [Recall.ai Bot Configuration docs](https://docs.recall.ai/reference/bot_create)
3. Common additions: `automatic_leave`, `video_mixed_layout`, `bot_name`, `deduplication_key`

#### Handling New Webhook Event Types
1. Add case in `webhook.ts` handler
2. Check event structure in Recall.ai docs
3. Consider whether to store in DB or just process

#### Adding Database Fields
1. Update `prisma/schema.prisma`
2. Run `npx prisma migrate dev -n <migration_name>`
3. Update TypeScript types (Prisma auto-generates)

## Important Conventions

### API Key Usage
- Always use raw key: `Authorization: ${process.env.RECALL_API_KEY}`
- Never add "Bearer " prefix (Recall.ai expects raw key)
- Store in `.env` as `RECALL_API_KEY`

### Error Handling
- Webhook handlers should ACK quickly (within 5s)
- Log errors but don't fail request
- Use try-catch for DB operations
- Fetch errors should be logged with full context

### Polling Patterns
- `waitForRecordingId()`: 3s intervals, 20 tries max
- `fetchArtifacts()`: 5s intervals, 10 tries max for each media type
- Frontend: Polls `/api/userData` every 3-5s after starting bot

### WebSocket Broadcasting
- All transcript updates broadcast to all clients
- Consider adding room/channel logic for multi-meeting support
- WebSocket server runs independently of Next.js process

## Common Tasks

### Adding a New Real-Time Event Handler
```typescript
// In webhook.ts
if (data.event === 'new_event_type') {
  const payload = data.data
  // Process payload
  // Save to DB if needed
  // Broadcast via WebSocket if needed
}
```

### Retrieving Additional Recording Artifacts
```typescript
// In lib/recall-media.ts
export async function fetchNewArtifact(recordingId: string) {
  const url = `${BASE}/new_artifact?recording_id=${recordingId}`
  const r = await fetch(url, { headers: HEADERS })
  if (!r.ok) return null
  const data = await r.json()
  return data?.results?.[0]?.data?.download_url
}
```

### Adding Meeting Metadata
```typescript
// In startRecall.ts, add to bot creation:
metadata: {
  external_id: externalId,
  user_name: userName,
  meeting_title: title,
  // Custom fields here
}
```

## Gotchas and Known Issues

1. **ngrok Static Domain Required**: Free tier only allows one static domain. Webhook URL must match your configured domain.

2. **WebSocket CORS**: If running browser from different origin, update WebSocket server CORS settings.

3. **Prisma Generate**: Run `npx prisma generate` after any schema changes before running app.

4. **PostgreSQL Must Be Running**: App will crash on startup if Postgres isn't accessible.

5. **Media Availability Timing**: After `done` event, media URLs may take 5-10s to appear. The `fetchArtifacts()` function handles this with retries.

6. **Transcript Provider**: Currently set to `meeting_captions` (native Zoom/Google Meet captions). Can switch to `assembly_ai` or `deepgram` for better quality (requires credits).

7. **Bot Admission**: Bot must be manually admitted to Zoom calls. Waiting room must be bypassed or someone must admit the bot.

8. **Region Selection**: Base URL includes region (`us-west-2`). Ensure your Recall.ai workspace matches the region in code.

## Testing Checklist

### End-to-End Flow
- [ ] Start Next.js dev server
- [ ] Start WebSocket server
- [ ] Start ngrok tunnel
- [ ] Create Zoom meeting
- [ ] Submit Zoom link via UI
- [ ] Check bot created in Recall.ai dashboard
- [ ] Admit bot to Zoom meeting
- [ ] Verify real-time transcripts appear in browser
- [ ] Speak for 30s to generate transcript data
- [ ] End Zoom call
- [ ] Check webhook logs for `call_ended` and `done` events
- [ ] Wait for media URLs to populate in UI
- [ ] Verify MP4 video downloads
- [ ] Verify MP3 audio downloads
- [ ] Check PostgreSQL for Meeting and Transcript records

### Database Verification
```sql
-- Check latest meeting
SELECT * FROM "Meeting" ORDER BY "createdAt" DESC LIMIT 1;

-- Check transcripts for meeting
SELECT * FROM "Transcript" WHERE "meetingId" = '<id>' ORDER BY "timestamp";
```

## Useful Resources

- [Recall.ai Documentation](https://docs.recall.ai)
- [Recall.ai API Reference](https://docs.recall.ai/reference)
- [Next.js API Routes](https://nextjs.org/docs/api-routes/introduction)
- [Prisma Documentation](https://www.prisma.io/docs)
- [WebSocket API (ws)](https://github.com/websockets/ws)

## Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@localhost:5432/recall_demo` |
| `RECALL_API_KEY` | Recall.ai API authentication | `abc123xyz456` (no "Bearer ") |

## Quick Commands

```bash
# Install dependencies
pnpm install

# Generate Prisma client
npx prisma generate

# Run database migrations
npx prisma migrate dev

# Start Next.js dev server
npm run dev

# Start WebSocket server
npx ts-node ws-server.ts

# Start ngrok tunnels
ngrok start --all

# View database in browser
npx prisma studio

# Reset database (careful!)
npx prisma migrate reset
```
