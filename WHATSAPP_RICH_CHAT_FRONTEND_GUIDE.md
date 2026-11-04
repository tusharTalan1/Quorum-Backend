# WhatsApp-Like Rich Chat Features — Frontend Integration Guide

This guide provides end-to-end specifications, payload schemas, Socket.IO event listeners, and REST API endpoints for all WhatsApp-style rich chat features across Quorum (Direct Messages, Project collaboration chats, Team chats, Custom Group chats, and Community channels).

---

## Table of Contents
1. [Core Message Data Model](#1-core-message-data-model)
2. [Socket.IO Connection & Setup](#2-socketio-connection--setup)
3. [Feature 1: Emoji Reactions](#3-feature-1-emoji-reactions)
4. [Feature 2: Quoted Replies](#4-feature-2-quoted-replies)
5. [Feature 3: Voice Messages & Waveforms](#5-feature-3-voice-messages--waveforms)
6. [Feature 4: Message Editing](#6-feature-4-message-editing)
7. [Feature 5: Message Forwarding](#7-feature-5-message-forwarding)
8. [Feature 6: Message Deletion ("Delete for Everyone" vs "Delete for Me")](#8-feature-6-message-deletion)
9. [Feature 7: WhatsApp-Style Polls](#9-feature-7-whatsapp-style-polls)
10. [REST API Reference](#10-rest-api-reference)
11. [Frontend TypeScript Type Definitions](#11-frontend-typescript-type-definitions)

---

## 1. Core Message Data Model

Every message returned by REST APIs and Socket.IO events adheres to this structure:

```json
{
  "id": "c1f727c9-4f76-432d-9473-b26f53488dc4",
  "conversationId": "3b290940-1bc3-4876-b6f7-495ccfa51bf1",
  "senderId": "user-uuid-1",
  "content": "Check out this update!",
  "fileUrl": "https://s3.amazonaws.com/quorum/conversations/conv-1/voice.m4a",
  "fileType": "voice",
  "fileName": "voice_note.m4a",
  "fileSize": 65536,
  "audioDuration": 8,
  "audioWaveform": [0.12, 0.45, 0.85, 0.65, 0.3, 0.95, 0.5, 0.2],
  "clientMessageId": "temp-uuid-client-123",
  "seq": 42,
  "mentionedUserIds": [],
  "replyToId": "msg-original-uuid",
  "replyTo": {
    "id": "msg-original-uuid",
    "content": "Can we review this deliverable?",
    "fileUrl": null,
    "fileType": null,
    "fileName": null,
    "audioDuration": null,
    "isDeleted": false,
    "sender": {
      "id": "user-uuid-2",
      "name": "Sarah Connor",
      "avatar": "https://res.cloudinary.com/..."
    },
    "poll": {
      "id": "poll-uuid-1",
      "question": "Which design mock do we prefer?"
    }
  },
  "isEdited": false,
  "editedAt": null,
  "isForwarded": false,
  "forwardCount": 0,
  "isDeleted": false,
  "deletedAt": null,
  "reactions": [
    {
      "emoji": "👍",
      "count": 2,
      "users": [
        { "id": "user-uuid-2", "name": "Sarah Connor", "avatar": "https://..." },
        { "id": "user-uuid-3", "name": "Alex Smith", "avatar": null }
      ]
    }
  ],
  "poll": {
    "id": "poll-uuid-1",
    "question": "Which tech stack should we pick for the project?",
    "allowMultipleAnswers": false,
    "createdAt": "2026-11-04T21:40:00.000Z",
    "totalVotes": 3,
    "uniqueVotersCount": 3,
    "hasVoted": true,
    "userVotedOptionIds": ["opt-uuid-1"],
    "options": [
      {
        "id": "opt-uuid-1",
        "text": "Next.js + TypeScript",
        "order": 0,
        "voteCount": 2,
        "hasVoted": true,
        "voters": [
          {
            "userId": "user-uuid-1",
            "user": { "id": "user-uuid-1", "name": "Jane Doe", "avatar": "https://res.cloudinary.com/..." },
            "createdAt": "2026-11-04T21:41:00.000Z"
          }
        ]
      },
      {
        "id": "opt-uuid-2",
        "text": "Flutter",
        "order": 1,
        "voteCount": 1,
        "hasVoted": false,
        "voters": []
      }
    ]
  },
  "createdAt": "2026-11-04T10:00:00.000Z",
  "updatedAt": "2026-11-04T10:00:00.000Z",
  "sender": {
    "id": "user-uuid-1",
    "name": "Jane Doe",
    "avatar": "https://res.cloudinary.com/..."
  }
}
```

---

## 2. Socket.IO Connection & Setup

```javascript
import { io } from "socket.io-client";

const socket = io("https://api.newquorum.me", {
  auth: {
    token: accessToken // JWT Bearer token
  },
  transports: ["websocket", "polling"],
  withCredentials: true
});

// Join conversation room when entering chat screen
socket.emit("conversation:join", { conversationId: "conv-uuid" }, (res) => {
  if (res.success) console.log("Joined conversation room:", res.data.conversationId);
});

// Leave conversation room when exiting chat screen
socket.emit("conversation:leave", { conversationId: "conv-uuid" });
```

---

## 3. Feature 1: Emoji Reactions

### Behavior
- Any participant can add an emoji reaction (`👍`, `❤️`, `😂`, `😮`, `😢`, `🙏`, etc.) to any active message.
- **WhatsApp Toggle Logic**: Sending the same emoji that the user already reacted with will remove their reaction. Sending a different emoji adds/updates it.
- Deleted messages cannot be reacted to.

### Client → Server Event: `message:react`
```javascript
socket.emit("message:react", {
  conversationId: "conv-uuid",
  messageId: "msg-uuid",
  emoji: "❤️"
}, (response) => {
  if (response.success) {
    // response.data contains updated reactions list
    updateLocalMessageReactions(response.data.messageId, response.data.reactions);
  }
});
```

### Server → Client Broadcast: `message:reaction_updated`
Broadcasted to all participants in the conversation room:
```javascript
socket.on("message:reaction_updated", (payload) => {
  const { conversationId, messageId, emoji, userId, action, reactions } = payload;
  // action: 'added' | 'removed'
  // reactions: array of grouped { emoji, count, users: [{ id, name, avatar }] }
  updateMessageReactionsInUI(messageId, reactions);
});
```

### REST API Fallback
- `POST /api/conversations/:id/messages/:messageId/reactions`
  - Body: `{ "emoji": "❤️" }`
  - Response (200 OK):
    ```json
    {
      "success": true,
      "message": "Reaction updated successfully",
      "data": {
        "messageId": "msg-uuid",
        "conversationId": "conv-uuid",
        "emoji": "❤️",
        "userId": "user-uuid-1",
        "action": "added",
        "reactions": [
          {
            "emoji": "❤️",
            "count": 1,
            "users": [{ "id": "user-uuid-1", "name": "Jane", "avatar": null }]
          }
        ]
      }
    }
    ```

---

## 4. Feature 2: Quoted Replies

### Behavior
- Swiping right or clicking "Reply" on a message sets `replyToId` in the message composer.
- Shows a preview card above the input box (sender name, snippet of content, or media indicator).
- When sent, the backend validates that `replyToId` belongs to the same conversation and populates the `replyTo` object with the original sender and media preview.

### Client → Server Event: `message:send`
```javascript
socket.emit("message:send", {
  conversationId: "conv-uuid",
  clientMessageId: "temp-reply-1234",
  content: "I agree with your suggestion above!",
  replyToId: "msg-original-uuid"
}, (response) => {
  if (response.success) {
    replaceOptimisticMessage(response.data);
  }
});
```

### Rendering Quoted Replies in Chat Bubbles
When `message.replyTo` is present, render the reply banner inside the bubble:
- If `replyTo.isDeleted` is true, show `"This message was deleted"`.
- If `replyTo.fileType === "voice"`, show a microphone icon + duration (`${replyTo.audioDuration}s Voice message`).
- If `replyTo.fileType === "image"`, show a small thumbnail preview.
- Clicking the reply banner smoothly scrolls the chat list to `replyTo.id`.

---

## 5. Feature 3: Voice Messages & Waveforms

### Behavior
- Record audio via mobile microphone (React Native / Flutter / Web Audio MediaRecorder).
- While recording, collect audio duration in seconds and sample normalized sound amplitudes (`0.0` to `1.0`) into an array of up to 64–128 numbers for the waveform.
- Request presigned S3 upload URL via `POST /api/uploads/sign` with `fileType: "voice"`.
- Upload the raw audio blob to S3 via HTTP `PUT`.
- Send the message with `fileUrl`, `fileType: "voice"`, `audioDuration`, and `audioWaveform`.

### Step 1: Request Presigned Upload URL
```javascript
const response = await fetch("https://api.newquorum.me/api/uploads/sign", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${accessToken}`
  },
  body: JSON.stringify({
    conversationId: "conv-uuid",
    fileName: "voice_note.m4a", // or .mp3, .wav, .webm, .ogg
    fileSize: audioBlob.size,
    fileType: "voice"
  })
});
const { data } = await response.json();
// data.uploadUrl -> S3 presigned PUT URL
// data.fileUrl   -> permanent file URL
```

### Step 2: Upload Audio Blob Directly to S3
```javascript
await fetch(data.uploadUrl, {
  method: "PUT",
  headers: {
    "Content-Type": data.metadata.contentType || "audio/mp4"
  },
  body: audioBlob
});
```

### Step 3: Send Voice Message via Socket
```javascript
socket.emit("message:send", {
  conversationId: "conv-uuid",
  clientMessageId: "temp-voice-uuid",
  content: "",
  fileUrl: data.fileUrl,
  fileType: "voice",
  fileName: "voice_note.m4a",
  fileSize: audioBlob.size,
  audioDuration: Math.round(durationInSeconds), // e.g., 14
  audioWaveform: normalizedWaveformArray        // e.g., [0.1, 0.4, 0.8, 0.6, ...]
}, (ack) => {
  if (ack.success) {
    replaceOptimisticVoiceMessage(ack.data);
  }
});
```

### Waveform UI Guidelines
- Render vertical bars corresponding to `message.audioWaveform`.
- Highlight bars proportionally as audio plays back from `0:00` to `audioDuration`.
- Display a Play/Pause button and formatted duration timer (`0:14`).

---

## 6. Feature 4: Message Editing

### Behavior
- Only the original sender can edit their message.
- Edits are permitted within 24 hours of creation.
- Deleted messages cannot be edited.
- Once edited, `isEdited: true` and `editedAt` are set. The frontend displays an `(edited)` badge beside the message timestamp.

### Client → Server Event: `message:edit`
```javascript
socket.emit("message:edit", {
  conversationId: "conv-uuid",
  messageId: "msg-uuid",
  content: "Corrected message text"
}, (response) => {
  if (response.success) {
    updateMessageContent(response.data);
  }
});
```

### Server → Client Broadcast: `message:updated`
```javascript
socket.on("message:updated", (payload) => {
  const { id, conversationId, content, isEdited, editedAt } = payload;
  updateMessageContent({ id, content, isEdited, editedAt });
});
```

### REST API Fallback
- `PATCH /api/conversations/:id/messages/:messageId`
  - Body: `{ "content": "Corrected message text" }`
  - Response (200 OK):
    ```json
    {
      "success": true,
      "message": "Message edited successfully",
      "data": {
        "id": "msg-uuid",
        "content": "Corrected message text",
        "isEdited": true,
        "editedAt": "2026-11-04T10:05:00.000Z"
      }
    }
    ```

---

## 7. Feature 5: Message Forwarding

### Behavior
- Users can select one or more messages (text, media, or voice notes) and forward them to one or more conversations simultaneously.
- Preserves the exact selection order of forwarded messages.
- Forwarded messages are marked with `isForwarded: true` and incremented `forwardCount`.
- Forwarded voice notes maintain their `audioDuration` and `audioWaveform`.

### Client → Server Event: `message:forward`
```javascript
socket.emit("message:forward", {
  messageIds: ["msg-uuid-1", "msg-uuid-2"],
  targetConversationIds: ["target-conv-1", "target-conv-2"]
}, (response) => {
  if (response.success) {
    showNotification("Messages forwarded successfully");
  }
});
```

### Real-Time Delivery
- Each target conversation room receives standard `message:new` events for the forwarded messages.
- Forwarded messages render with a `"Forwarded"` label (and forward icon) at the top of the bubble.

### REST API Fallback
- `POST /api/conversations/messages/forward`
  - Body:
    ```json
    {
      "messageIds": ["msg-uuid-1", "msg-uuid-2"],
      "targetConversationIds": ["target-conv-1", "target-conv-2"]
    }
    ```
  - Response (200 OK): Array of newly created message objects.

---

## 8. Feature 6: Message Deletion

### Two Modes:
1. **Delete for Everyone (`deleteForEveryone: true`)**:
   - Allowed for message sender or group owner/admin.
   - Soft-deletes the message globally.
   - Content is replaced with `"This message was deleted"`.
   - Attachments, voice audio URLs, and waveform data are cleared.
   - Reply chains remain intact without breaking references.
   - Broadcasted to all participants in the conversation room.
2. **Delete for Me (`deleteForEveryone: false`)**:
   - Allowed for any participant in the conversation.
   - Hides the message solely from the requesting user's view and unread counts.
   - Other participants can still see and interact with the message.
   - Broadcasted solely to `user:${userId}`.

### Client → Server Event: `message:delete`
```javascript
// Delete for Everyone
socket.emit("message:delete", {
  conversationId: "conv-uuid",
  messageId: "msg-uuid",
  deleteForEveryone: true
}, (response) => {
  if (response.success) {
    markMessageAsDeletedInUI(response.data.messageId);
  }
});

// Delete for Me
socket.emit("message:delete", {
  conversationId: "conv-uuid",
  messageId: "msg-uuid",
  deleteForEveryone: false
}, (response) => {
  if (response.success) {
    removeMessageFromMyLocalList(response.data.messageId);
  }
});
```

### Server → Client Broadcast: `message:deleted`
```javascript
socket.on("message:deleted", (payload) => {
  const { conversationId, messageId, isDeleted, deleteForEveryone, deletedForMe, message } = payload;
  if (deleteForEveryone) {
    // Replace content in UI with "This message was deleted"
    renderDeletedPlaceholder(messageId);
  } else if (deletedForMe) {
    // Remove the message entirely from current user's local screen
    removeMessageFromScreen(messageId);
  }
});
```

### REST API Fallback
- `DELETE /api/conversations/:id/messages/:messageId`
  - Body (Optional): `{ "deleteForEveryone": true }` or `{ "deleteForEveryone": false }`
  - Response (200 OK):
    ```json
    {
      "success": true,
      "message": "Message deleted successfully",
      "data": {
        "messageId": "msg-uuid",
        "conversationId": "conv-uuid",
        "isDeleted": true,
        "deleteForEveryone": true
      }
    }
    ```

---

## 9. Feature 7: WhatsApp-Style Polls

WhatsApp-like Polls are supported across all conversations (Direct Messages, Project collaboration chats, Team chats, Custom Group chats, and Channels).

### Key Rules & Behavior
1. **Option Limits:** Each poll contains between 2 and 12 options.
2. **Single-Choice vs Multi-Choice (`allowMultipleAnswers`):**
   - Single-choice (`false`): Voting for an option clears any previous vote and assigns the vote to the new option. Clicking the already selected option unvotes.
   - Multi-choice (`true`): Clicking an option toggles that option independently (adds if not voted, removes if voted).
3. **Real-Time Live Tallies:**
   - Any vote cast or toggled broadcasts `poll:updated` to all active participants in the conversation room.
   - The payload contains full option tallies, total votes, unique voters count, and the complete voter list.
4. **Forwarding Polls:**
   - When a message containing a poll is forwarded, the target conversation receives a clean new clone of the poll with all options and 0 votes.

### A. Create Poll

#### 1. Via Socket.IO (`message:send`)
```javascript
socket.emit(
  "message:send",
  {
    conversationId: "conv-uuid",
    poll: {
      question: "Which sprint deliverable should we prioritize?",
      options: ["Auth flow", "Analytics dashboard", "Payment gateway"],
      allowMultipleAnswers: false
    }
  },
  (response) => {
    if (response.success) {
      console.log("Poll created:", response.data.poll);
    }
  }
);
```

#### 2. Via REST API
- `POST /api/conversations/:id/messages/poll`
- **Body:**
  ```json
  {
    "question": "Which sprint deliverable should we prioritize?",
    "options": ["Auth flow", "Analytics dashboard", "Payment gateway"],
    "allowMultipleAnswers": false,
    "replyToId": "optional-reply-target-uuid"
  }
  ```
- **Response (201 Created):**
  Returns the new `ChatMessage` with `poll` populated.

---

### B. Vote on Poll

#### 1. Via Socket.IO (`poll:vote`)
```javascript
socket.emit(
  "poll:vote",
  {
    conversationId: "conv-uuid",
    messageId: "msg-uuid-poll",
    optionIds: ["opt-uuid-1"]
  },
  (response) => {
    if (response.success) {
      console.log("Updated caller poll state:", response.data.poll);
    }
  }
);
```

#### 2. Via REST API
- `POST /api/conversations/:id/messages/:messageId/poll/vote` (or `POST /api/conversations/:id/polls/:pollId/vote`)
- **Body:**
  ```json
  {
    "optionIds": ["opt-uuid-1"]
  }
  ```
- **Response (200 OK):**
  Returns `{ success: true, conversationId, messageId, pollId, poll }`.

---

### C. Live Real-Time Listener (`poll:updated`)
```javascript
socket.on("poll:updated", ({ conversationId, messageId, pollId, poll }) => {
  updateMessagePoll(messageId, poll);
});
```

---

## 10. REST API Reference

| Method | Endpoint | Description | Key Body Parameters |
|---|---|---|---|
| `POST` | `/api/uploads/sign` | Get presigned upload URL for media or voice notes | `fileName`, `fileSize`, `fileType: "voice" \| "audio" \| "image" \| "file"`, `conversationId` |
| `POST` | `/api/conversations/:id/messages/poll` | Create a poll in conversation | `question: string`, `options: string[]`, `allowMultipleAnswers?: boolean` |
| `POST` | `/api/conversations/:id/messages/:messageId/poll/vote` | Vote or toggle poll option by messageId | `optionIds: string[]` |
| `POST` | `/api/conversations/:id/polls/:pollId/vote` | Vote or toggle poll option by pollId | `optionIds: string[]` |
| `POST` | `/api/conversations/:id/messages/:messageId/reactions` | Toggle emoji reaction | `emoji` (string) |
| `POST` | `/api/conversations/messages/forward` | Forward messages to chats | `messageIds: string[]`, `targetConversationIds: string[]` |
| `PATCH` | `/api/conversations/:id/messages/:messageId` | Edit message content (within 24h) | `content` (string) |
| `DELETE` | `/api/conversations/:id/messages/:messageId` | Delete message | `deleteForEveryone: boolean` (optional, default true) |
| `GET` | `/api/conversations/:id/messages` | Fetch paginated chat messages | Query: `cursor`, `limit` (default 20, max 50) |
| `GET` | `/api/conversations/:id/search` | Full-text search within chat | Query: `q`, `cursor`, `limit` |

---

## 11. Frontend TypeScript Type Definitions

Copy and paste these interfaces directly into your frontend code (`types/chat.ts`):

```typescript
export interface MessageReactionUser {
  id: string;
  name: string | null;
  avatar: string | null;
}

export interface MessageReactionGroup {
  emoji: string;
  count: number;
  users: MessageReactionUser[];
}

export interface PollVoter {
  userId: string;
  user: {
    id: string;
    name: string | null;
    avatar: string | null;
  } | null;
  createdAt: string;
}

export interface PollOption {
  id: string;
  text: string;
  order: number;
  voteCount: number;
  hasVoted?: boolean;
  voters: PollVoter[];
}

export interface Poll {
  id: string;
  question: string;
  allowMultipleAnswers: boolean;
  createdAt: string;
  totalVotes: number;
  uniqueVotersCount: number;
  hasVoted?: boolean;
  userVotedOptionIds?: string[];
  options: PollOption[];
}

export interface QuotedReplyPreview {
  id: string;
  content: string;
  fileUrl: string | null;
  fileType: string | null;
  fileName: string | null;
  audioDuration: number | null;
  isDeleted: boolean;
  sender: {
    id: string;
    name: string | null;
    avatar: string | null;
  };
  poll?: {
    id: string;
    question: string;
  } | null;
}

export interface ChatMessage {
  id: string;
  conversationId: string;
  senderId: string;
  content: string;
  fileUrl: string | null;
  fileType: 'image' | 'video' | 'audio' | 'voice' | 'file' | 'document' | 'poll' | null;
  fileName: string | null;
  fileSize: number | null;
  audioDuration: number | null;
  audioWaveform: number[];
  clientMessageId: string | null;
  seq: number;
  mentionedUserIds: string[];
  replyToId: string | null;
  replyTo: QuotedReplyPreview | null;
  isEdited: boolean;
  editedAt: string | null;
  isForwarded: boolean;
  forwardCount: number;
  isDeleted: boolean;
  deletedAt: string | null;
  reactions: MessageReactionGroup[];
  poll: Poll | null;
  createdAt: string;
  updatedAt: string;
  sender: {
    id: string;
    name: string | null;
    avatar: string | null;
  };
}

export interface SendMessagePayload {
  conversationId: string;
  clientMessageId?: string;
  content?: string;
  fileUrl?: string;
  fileType?: string;
  fileName?: string;
  fileSize?: number;
  audioDuration?: number;
  audioWaveform?: number[];
  replyToId?: string;
  mentions?: string[];
  poll?: {
    question: string;
    options: string[];
    allowMultipleAnswers?: boolean;
  };
}

export interface VotePollPayload {
  conversationId: string;
  messageId?: string;
  pollId?: string;
  optionIds: string[];
}
```
