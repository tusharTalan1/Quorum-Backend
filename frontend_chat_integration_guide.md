# Real-Time Chat & Direct Messaging API Integration Guide

This guide covers everything required to integrate the Chat & Direct Messaging features in the Quorum frontend (Web & Mobile).

---

## 1. Socket.IO Connection & Authentication

### Connection Setup
Connect to the Socket.IO server passing the JWT Access Token in `auth.token` or the `Authorization` header:

```javascript
import { io } from "socket.io-client";

const socket = io("https://api.newquorum.me", {
  auth: {
    token: accessToken // e.g. from user session or login
  },
  transports: ["websocket", "polling"],
  withCredentials: true
});

socket.on("connect", () => {
  console.log("Connected to Quorum socket server:", socket.id);
});

socket.on("connect_error", (err) => {
  console.error("Socket connection error:", err.message);
});
```

---

## 2. Socket.IO Events Reference

### A. Client → Server Events

#### 1. `conversation:join`
Join a specific conversation's real-time room to receive active typing and message events.
* **Payload:**
  ```json
  {
    "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821"
  }
  ```
* **Acknowledgement Callback:**
  ```json
  { "success": true }
  ```

#### 2. `conversation:leave`
Leave a conversation room when navigating away from the chat screen.
* **Payload:**
  ```json
  {
    "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821"
  }
  ```
* **Acknowledgement Callback:**
  ```json
  { "success": true }
  ```

#### 3. `message:send`
Send a new text, media, or voice message in a conversation. Supports optimistic UI via `clientMessageId`, quoted replies via `replyToId`, and voice notes via `audioDuration` and `audioWaveform`.
* **Text / Media Payload:**
  ```json
  {
    "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821",
    "clientMessageId": "temp-uuid-12345",
    "content": "Sounds great! Let's do that.",
    "fileUrl": "https://res.cloudinary.com/quorum/image/upload/sample.jpg",
    "fileType": "image",
    "replyToId": "msg-original-uuid-123"
  }
  ```
* **Voice Message Payload:**
  ```json
  {
    "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821",
    "clientMessageId": "temp-uuid-voice-123",
    "content": "",
    "fileUrl": "https://s3.amazonaws.com/quorum-bucket/conversations/conv-123/voice.m4a",
    "fileType": "voice",
    "fileName": "voice_note.m4a",
    "audioDuration": 8,
    "audioWaveform": [0.15, 0.45, 0.8, 0.65, 0.3, 0.9, 0.55, 0.2]
  }
  ```
* **Acknowledgement Callback:**
  ```json
  {
    "success": true,
    "data": {
      "id": "msg-9f20e4b8-2e0f-4df4-a829-d591b9204391",
      "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821",
      "senderId": "user-uuid-123",
      "content": "",
      "fileUrl": "https://s3.amazonaws.com/quorum-bucket/conversations/conv-123/voice.m4a",
      "fileType": "voice",
      "fileName": "voice_note.m4a",
      "fileSize": 65536,
      "audioDuration": 8,
      "audioWaveform": [0.15, 0.45, 0.8, 0.65, 0.3, 0.9, 0.55, 0.2],
      "seq": 42,
      "replyToId": null,
      "replyTo": null,
      "isEdited": false,
      "editedAt": null,
      "isForwarded": false,
      "forwardCount": 0,
      "isDeleted": false,
      "reactions": [],
      "createdAt": "2026-10-09T18:00:00.000Z",
      "sender": {
        "id": "user-uuid-123",
        "name": "Jane Doe",
        "avatar": "https://res.cloudinary.com/..."
      }
    }
  }
  ```

#### 4. `message:read`
Mark a specific message as read. Advances the user's read boundary up to that message sequence number.
* **Payload:**
  ```json
  {
    "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821",
    "messageId": "msg-9f20e4b8-2e0f-4df4-a829-d591b9204391"
  }
  ```
* **Acknowledgement Callback:**
  ```json
  {
    "success": true,
    "data": {
      "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821",
      "userId": "reader-user-uuid",
      "lastReadSeq": 42,
      "lastReadMessageId": "msg-9f20e4b8-2e0f-4df4-a829-d591b9204391",
      "updated": true
    }
  }
  ```

#### 5. `typing:start` & `typing:stop`
Notify participants that the current user started or stopped typing. (Debounce on keypress and stop after 1.5–2s of inactivity).
* **Payload:**
  ```json
  {
    "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821"
  }
  ```

#### 6. `message:edit`
Edit a previously sent message within the 24-hour edit window (sender-only).
* **Payload:**
  ```json
  {
    "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821",
    "messageId": "msg-9f20e4b8-2e0f-4df4-a829-d591b9204391",
    "content": "Updated message content here"
  }
  ```
* **Acknowledgement Callback:**
  ```json
  {
    "success": true,
    "data": {
      "id": "msg-9f20e4b8-2e0f-4df4-a829-d591b9204391",
      "content": "Updated message content here",
      "isEdited": true,
      "editedAt": "2026-10-09T18:05:00.000Z"
    }
  }
  ```

#### 7. `message:delete`
Delete a message for everyone (or sender). Masks deleted text with `"This message was deleted"` and preserves reply chain hierarchy.
* **Payload:**
  ```json
  {
    "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821",
    "messageId": "msg-9f20e4b8-2e0f-4df4-a829-d591b9204391",
    "deleteForEveryone": true
  }
  ```
* **Acknowledgement Callback:**
  ```json
  {
    "success": true,
    "data": {
      "id": "msg-9f20e4b8-2e0f-4df4-a829-d591b9204391",
      "content": "This message was deleted",
      "isDeleted": true,
      "deletedAt": "2026-10-09T18:06:00.000Z"
    }
  }
  ```

#### 8. `message:react`
Toggle an emoji reaction on any message. Sending the same emoji again removes the reaction (WhatsApp-style toggle).
* **Payload:**
  ```json
  {
    "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821",
    "messageId": "msg-9f20e4b8-2e0f-4df4-a829-d591b9204391",
    "emoji": "👍"
  }
  ```
* **Acknowledgement Callback:**
  ```json
  {
    "success": true,
    "data": {
      "messageId": "msg-9f20e4b8-2e0f-4df4-a829-d591b9204391",
      "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821",
      "reactions": [
        {
          "emoji": "👍",
          "count": 2,
          "users": [
            { "id": "user-uuid-1", "name": "Jane", "avatar": "https://..." },
            { "id": "user-uuid-2", "name": "Bob", "avatar": null }
          ]
        }
      ]
    }
  }
  ```

#### 9. `message:forward`
Forward one or more messages to one or more target conversations simultaneously. Automatically clones contents and attachments with `isForwarded: true` and incremented `forwardCount`.
* **Payload:**
  ```json
  {
    "messageIds": ["msg-uuid-1", "msg-uuid-2"],
    "targetConversationIds": ["conv-uuid-target-a", "conv-uuid-target-b"]
  }
  ```
* **Acknowledgement Callback:**
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "msg-new-clone-uuid",
        "conversationId": "conv-uuid-target-a",
        "content": "Original message text",
        "isForwarded": true,
        "forwardCount": 1
      }
    ]
  }
  ```

#### 10. `poll:vote`
Cast or toggle vote on one or more options of a poll in a conversation.
* **Payload:**
  ```json
  {
    "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821",
    "messageId": "msg-uuid-poll-1",
    "optionIds": ["opt-uuid-1"]
  }
  ```
  *(Note: You can pass either `messageId` or `pollId`).*
* **Acknowledgement Callback:**
  ```json
  {
    "success": true,
    "data": {
      "success": true,
      "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821",
      "messageId": "msg-uuid-poll-1",
      "pollId": "poll-uuid-1",
      "poll": {
        "id": "poll-uuid-1",
        "question": "Which framework should we use?",
        "allowMultipleAnswers": false,
        "totalVotes": 3,
        "uniqueVotersCount": 3,
        "hasVoted": true,
        "userVotedOptionIds": ["opt-uuid-1"],
        "options": [
          {
            "id": "opt-uuid-1",
            "text": "React",
            "order": 0,
            "voteCount": 2,
            "hasVoted": true,
            "voters": [
              {
                "userId": "user-uuid-1",
                "user": { "id": "user-uuid-1", "name": "Alice", "avatar": "https://..." },
                "createdAt": "2026-11-04T21:40:00.000Z"
              }
            ]
          },
          {
            "id": "opt-uuid-2",
            "text": "Vue",
            "order": 1,
            "voteCount": 1,
            "hasVoted": false,
            "voters": []
          }
        ]
      }
    }
  }
  ```

---

### B. Server → Client Broadcast Events

#### 1. `message:new`
Broadcasted to the conversation room when a new message is posted or forwarded.
* **Payload:**
  ```json
  {
    "id": "msg-uuid",
    "conversationId": "conv-uuid",
    "senderId": "user-uuid",
    "content": "Hello!",
    "fileUrl": null,
    "fileType": null,
    "replyToId": "msg-reply-target-uuid",
    "replyTo": {
      "id": "msg-reply-target-uuid",
      "content": "Original message text",
      "fileUrl": null,
      "audioDuration": null,
      "sender": { "id": "user-id", "name": "Original Sender", "avatar": "..." }
    },
    "audioDuration": null,
    "audioWaveform": [],
    "isEdited": false,
    "isForwarded": false,
    "forwardCount": 0,
    "reactions": [],
    "seq": 43,
    "createdAt": "2026-10-09T18:01:00.000Z",
    "sender": {
      "id": "user-uuid",
      "name": "Alex Smith",
      "avatar": "https://..."
    }
  }
  ```

#### 2. `message:updated`
Broadcasted to conversation participants when a sender edits a message.
* **Payload:**
  ```json
  {
    "id": "msg-uuid",
    "conversationId": "conv-uuid",
    "content": "Updated message content",
    "isEdited": true,
    "editedAt": "2026-10-09T18:05:00.000Z"
  }
  ```

#### 3. `message:deleted`
Broadcasted to conversation participants when a message is soft-deleted ("Delete for everyone").
* **Payload:**
  ```json
  {
    "id": "msg-uuid",
    "conversationId": "conv-uuid",
    "content": "This message was deleted",
    "fileUrl": null,
    "isDeleted": true,
    "deletedAt": "2026-10-09T18:06:00.000Z"
  }
  ```

#### 4. `message:reaction_updated`
Broadcasted to conversation participants when a reaction is added or removed.
* **Payload:**
  ```json
  {
    "messageId": "msg-uuid",
    "conversationId": "conv-uuid",
    "reactions": [
      {
        "emoji": "👍",
        "count": 1,
        "users": [
          { "id": "user-uuid-1", "name": "Jane", "avatar": "https://..." }
        ]
      }
    ]
  }
  ```

#### 5. `conversation:update`
Broadcasted to participants' user channels (`user:${userId}`) to update the conversation inbox preview (last message and unread count) in real time.
* **Payload:**
  ```json
  {
    "conversationId": "conv-uuid",
    "lastMessage": {
      "id": "msg-uuid",
      "content": "Hello!",
      "createdAt": "2026-10-09T18:01:00.000Z",
      "sender": { "id": "user-uuid", "name": "Alex Smith" }
    }
  }
  ```

#### 6. `message:read`
Broadcasted to the conversation when another participant reads messages.
* **Payload:**
  ```json
  {
    "conversationId": "conv-uuid",
    "userId": "reader-user-uuid",
    "lastReadSeq": 43,
    "lastReadMessageId": "msg-uuid",
    "messageId": "msg-uuid"
  }
  ```

#### 7. `typing:start` & `typing:stop`
Broadcasted when another participant starts or stops typing.
* **Payload:**
  ```json
  {
    "conversationId": "conv-uuid",
    "userId": "typing-user-uuid"
  }
  ```

#### 8. `conversation:removed`
Broadcasted to a participant's personal channel (`user:${userId}`) when they are removed from a group conversation by the group owner.
* **Payload:**
  ```json
  {
    "conversationId": "conv-uuid",
    "removedBy": "owner-user-uuid"
  }
  ```

#### 9. `presence:online` & `presence:offline`
Broadcasted to contacts/co-participants when a user connects or disconnects.
* **Online Payload:** `{ "userId": "user-uuid" }`
* **Offline Payload:** `{ "userId": "user-uuid", "lastSeen": "2026-10-09T18:05:00.000Z" }`

#### 10. `poll:updated`
Broadcasted in real-time to all conversation participants whenever any user casts, changes, or removes a vote on a poll.
* **Payload:**
  ```json
  {
    "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821",
    "messageId": "msg-uuid-poll-1",
    "pollId": "poll-uuid-1",
    "poll": {
      "id": "poll-uuid-1",
      "question": "Which framework should we use?",
      "allowMultipleAnswers": false,
      "totalVotes": 4,
      "uniqueVotersCount": 4,
      "options": [
        {
          "id": "opt-uuid-1",
          "text": "React",
          "order": 0,
          "voteCount": 3,
          "voters": [
            {
              "userId": "user-uuid-1",
              "user": { "id": "user-uuid-1", "name": "Alice", "avatar": "https://..." },
              "createdAt": "2026-11-04T21:40:00.000Z"
            }
          ]
        },
        {
          "id": "opt-uuid-2",
          "text": "Vue",
          "order": 1,
          "voteCount": 1,
          "voters": []
        }
      ]
    }
  }
  ```

---

## 3. REST API Endpoints

All endpoints require `Authorization: Bearer <ACCESS_TOKEN>`. For web clients using session cookies, include `x-xsrf-token` header if modifying data.

### 1. Start or Get Direct Conversation (1:1)
* **Endpoint:** `POST /api/dm`
* **Body:**
  ```json
  {
    "targetUserId": "user-uuid-to-chat-with"
  }
  ```
* **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": {
      "id": "conv-uuid",
      "type": "direct",
      "directKey": "userA_userB",
      "participants": [
        {
          "userId": "user-uuid-1",
          "user": { "id": "user-uuid-1", "name": "User 1", "avatar": "..." }
        },
        {
          "userId": "user-uuid-2",
          "user": { "id": "user-uuid-2", "name": "User 2", "avatar": "..." }
        }
      ]
    }
  }
  ```

### 2. Create Group Conversation
Create a new group chat with a name, optional avatar, and initial members. The creator automatically becomes the group owner (`ownerId`).
* **Endpoint:** `POST /api/conversations/group`
* **Body:**
  ```json
  {
    "name": "Project Engineering",
    "avatar": "https://res.cloudinary.com/...",
    "participantIds": [
      "user-uuid-1",
      "user-uuid-2"
    ]
  }
  ```
* **Response (201 Created):**
  ```json
  {
    "success": true,
    "data": {
      "id": "conv-uuid-group",
      "type": "group",
      "name": "Project Engineering",
      "avatar": "https://res.cloudinary.com/...",
      "ownerId": "creator-uuid",
      "participants": [
        { "userId": "creator-uuid", "role": "owner", "user": { "id": "creator-uuid", "name": "Creator" } },
        { "userId": "user-uuid-1", "role": "member", "user": { "id": "user-uuid-1", "name": "User 1" } },
        { "userId": "user-uuid-2", "role": "member", "user": { "id": "user-uuid-2", "name": "User 2" } }
      ]
    }
  }
  ```

### 3. Add Members to Group
Add one or more users to an existing group conversation.
* **Endpoint:** `POST /api/conversations/:id/participants`
* **Authorization Rules:** Requires participant membership in the group.
* **Body:**
  ```json
  {
    "participantIds": [
      "user-uuid-3",
      "user-uuid-4"
    ]
  }
  ```
* **Response (200 OK):** Returns updated group conversation with all active participants.
* **Real-time Event Emitted:** Broadcasts `conversation:update` to all added and existing participants' user channels (`user:${userId}`).

### 4. Remove Member or Leave Group
Leave a group (passing own `userId`) or remove another member (Owner only).
* **Endpoint:** `DELETE /api/conversations/:id/participants/:userId`
* **Authorization Rules:**
  - **Self-Removal (Leaving):** Any participant can remove themselves to leave the group. If the group owner leaves, ownership is automatically transferred to the next longest-standing group participant.
  - **Removing Others:** Only the group owner can remove other participants. Attempting to remove another user as a non-owner returns `403 Forbidden`.
* **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": {
      "success": true,
      "conversationDeleted": false,
      "newOwnerId": "next-owner-uuid-if-owner-left"
    }
  }
  ```
* **Real-time Events Emitted:** 
  - `conversation:removed` is emitted to the target user's personal channel `user:${userId}` so their UI immediately removes the conversation.
  - `conversation:update` is emitted to remaining participants.

### 5. Update Group Details (Name & Avatar)
Update group title or avatar picture.
* **Endpoint:** `PATCH /api/conversations/:id`
* **Authorization Rules:** Only the group owner can update group details.
* **Body:**
  ```json
  {
    "name": "Updated Group Title",
    "avatar": "https://res.cloudinary.com/..."
  }
  ```
* **Response (200 OK):** Returns updated group conversation object.
* **Real-time Event Emitted:** Broadcasts `conversation:update` to all participants.

### 6. List User Conversations
* **Endpoint:** `GET /api/conversations`
* **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "conv-uuid",
        "type": "direct",
        "unreadCount": 2,
        "lastMessage": {
          "id": "msg-uuid",
          "content": "Hey, let's connect!",
          "createdAt": "2026-10-09T17:50:00.000Z",
          "sender": { "id": "sender-uuid", "name": "Alex" }
        },
        "participants": [ ... ]
      }
    ]
  }
  ```

### 7. Fetch Paginated Messages (Cursor-based)
* **Endpoint:** `GET /api/conversations/:id/messages`
* **Query Parameters:**
  - `cursor`: (Optional) ID of the oldest message currently displayed.
  - `limit`: (Optional, default `20`, max `50`) Number of messages per page.
* **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": {
      "messages": [
        {
          "id": "msg-uuid-1",
          "conversationId": "conv-uuid",
          "senderId": "user-uuid-1",
          "content": "Sounds good!",
          "fileUrl": null,
          "fileType": null,
          "replyToId": "msg-prev-uuid",
          "replyTo": {
            "id": "msg-prev-uuid",
            "content": "Can we review the PR today?",
            "fileUrl": null,
            "sender": {
              "id": "user-uuid-2",
              "name": "Jane Doe",
              "avatar": "https://..."
            }
          },
          "isEdited": false,
          "editedAt": null,
          "isForwarded": true,
          "forwardCount": 1,
          "isDeleted": false,
          "reactions": [
            {
              "emoji": "👍",
              "count": 3,
              "users": [
                { "id": "user-uuid-2", "name": "Jane", "avatar": "https://..." }
              ]
            }
          ],
          "createdAt": "2026-10-09T18:00:00.000Z",
          "sender": {
            "id": "user-uuid-1",
            "name": "Alex",
            "avatar": "https://..."
          }
        }
      ],
      "nextCursor": "msg-uuid-oldest-in-this-page",
      "hasNextPage": true
    }
  }
  ```

### 8. Search Users (to Start a Chat)
* **Endpoint:** `GET /api/users/search` (or `GET /api/user/search`)
* **Query Parameters:**
  - `q`: Name or email query (string)
* **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "user-uuid-1",
        "name": "Jane Doe",
        "email": "jane@example.com",
        "avatar": "https://res.cloudinary.com/..."
      }
    ]
  }
  ```

### 9. Search Conversation Messages
* **Endpoint:** `GET /api/conversations/:id/search`
* **Query Parameters:**
  - `q`: Search keyword (string)
  - `cursor`: (Optional) Message ID cursor for pagination
  - `limit`: (Optional, default `20`)
* **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": {
      "messages": [ ... ],
      "nextCursor": "msg-uuid-next",
      "hasNextPage": false
    }
  }
  ```

### 10. Media Uploads (Cloudinary Signature)
* **Endpoint:** `POST /api/uploads/sign`
* **Body:**
  ```json
  {
    "folder": "chat_attachments"
  }
  ```
### 10. Media & Voice Uploads (Cloudinary / S3 Presigned URL)
* **Endpoint:** `POST /api/uploads/sign`
* **General / Cloudinary Upload Body:**
  ```json
  {
    "folder": "chat_attachments"
  }
  ```
* **Conversation File / Voice Note Upload Body:**
  ```json
  {
    "conversationId": "c4b9d031-18e4-4d45-933e-b490f2302821",
    "fileName": "voice_recording.m4a",
    "fileSize": 65536,
    "fileType": "voice"
  }
  ```
* **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": {
      "uploadUrl": "https://quorum-bucket.s3.amazonaws.com/conversations/conv-123/voice.m4a?AWSAccessKeyId=...",
      "fileUrl": "https://quorum-bucket.s3.amazonaws.com/conversations/conv-123/voice.m4a",
      "key": "conversations/conv-123/voice.m4a",
      "storageProvider": "s3",
      "metadata": {
        "fileName": "voice_recording.m4a",
        "fileSize": 65536,
        "fileType": "voice",
        "contentType": "audio/mp4"
      }
    }
  }
  ```
* **Voice Note Upload Flow:**
  1. Record audio on device (supports `.m4a`, `.mp3`, `.wav`, `.aac`, `.ogg`, `.opus`, `.webm`).
  2. Compute audio duration and sampled sound amplitudes for the waveform array.
  3. Call `POST /api/uploads/sign` with `fileType: "voice"` to receive `uploadUrl` and `fileUrl`.
  4. Perform HTTP `PUT` (or POST for Cloudinary) with the audio blob to `uploadUrl`.
  5. Emit `message:send` over Socket.IO passing `fileUrl`, `fileType: "voice"`, `audioDuration`, and `audioWaveform`.

### 11. FCM Push Device Tokens
* **Register Device:** `POST /api/devices`
  ```json
  {
    "token": "fcm_device_token_from_firebase",
    "platform": "android"
  }
  ```
* **Unregister Device (on Logout):** `DELETE /api/devices`
  ```json
  {
    "token": "fcm_device_token_from_firebase"
  }
  ```

### 12. Edit Message
Edit a previously sent message within the 24-hour edit window (sender-only).
* **Endpoint:** `PATCH /api/conversations/:id/messages/:messageId`
* **Body:**
  ```json
  {
    "content": "Updated message content here"
  }
  ```
* **Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Message edited successfully",
    "data": {
      "id": "msg-uuid",
      "content": "Updated message content here",
      "isEdited": true,
      "editedAt": "2026-10-09T18:05:00.000Z"
    }
  }
  ```

### 13. Delete Message ("Delete for everyone" vs "Delete for me")
* **Delete for everyone (`deleteForEveryone: true`):** Soft-deletes message for all participants, masks content with `"This message was deleted"`, clears attachment references, and preserves reply-to hierarchy (allowed for sender or admin).
* **Delete for me (`deleteForEveryone: false`):** Hides message only from the requesting user's view and unread counts without affecting any other conversation participants (allowed for any participant).
* **Endpoint:** `DELETE /api/conversations/:id/messages/:messageId`
* **Body (Optional):**
  ```json
  {
    "deleteForEveryone": true
  }
  ```
* **Response (200 OK):**
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

### 14. Toggle Message Reaction
Toggle emoji reaction on a message (WhatsApp style: adding an emoji again removes it).
* **Endpoint:** `POST /api/conversations/:id/messages/:messageId/reactions`
* **Body:**
  ```json
  {
    "emoji": "👍"
  }
  ```
* **Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Reaction updated successfully",
    "data": {
      "messageId": "msg-uuid",
      "conversationId": "conv-uuid",
      "reactions": [
        {
          "emoji": "👍",
          "count": 1,
          "users": [
            { "id": "user-uuid-1", "name": "Jane", "avatar": "https://..." }
          ]
        }
      ]
    }
  }
  ```

### 15. Forward Messages
Forward one or more messages to one or more conversations simultaneously.
* **Endpoint:** `POST /api/conversations/messages/forward`
* **Body:**
  ```json
  {
    "messageIds": ["msg-uuid-1", "msg-uuid-2"],
    "targetConversationIds": ["conv-uuid-target-a", "conv-uuid-target-b"]
  }
  ```
* **Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Messages forwarded successfully",
    "data": [
      {
        "id": "msg-new-clone-uuid",
        "conversationId": "conv-uuid-target-a",
        "content": "Original message text",
        "isForwarded": true,
        "forwardCount": 1
      }
    ]
  }
  ```

### 16. Create Poll in Conversation
Create a WhatsApp-like poll message across Direct Messages, Projects, Groups, Teams, or Channels.
* **Endpoint:** `POST /api/conversations/:id/messages/poll`
* **Body:**
  ```json
  {
    "question": "Which tech stack should we pick for the project?",
    "options": [
      "Next.js + TypeScript",
      "Vite + React",
      "Flutter"
    ],
    "allowMultipleAnswers": false,
    "replyToId": "optional-reply-target-msg-uuid"
  }
  ```
* **Response (201 Created):**
  ```json
  {
    "success": true,
    "message": "Poll created successfully",
    "data": {
      "id": "msg-uuid",
      "conversationId": "conv-uuid",
      "senderId": "user-uuid",
      "content": "Which tech stack should we pick for the project?",
      "fileType": "poll",
      "poll": {
        "id": "poll-uuid",
        "question": "Which tech stack should we pick for the project?",
        "allowMultipleAnswers": false,
        "totalVotes": 0,
        "uniqueVotersCount": 0,
        "hasVoted": false,
        "userVotedOptionIds": [],
        "options": [
          { "id": "opt-1", "text": "Next.js + TypeScript", "order": 0, "voteCount": 0, "hasVoted": false, "voters": [] },
          { "id": "opt-2", "text": "Vite + React", "order": 1, "voteCount": 0, "hasVoted": false, "voters": [] },
          { "id": "opt-3", "text": "Flutter", "order": 2, "voteCount": 0, "hasVoted": false, "voters": [] }
        ]
      }
    }
  }
  ```

### 17. Vote on Poll
Cast or toggle a vote on a poll option via REST. Supports voting by message ID (`/messages/:messageId/poll/vote`) or by poll ID (`/polls/:pollId/vote`).
* **Endpoint:** `POST /api/conversations/:id/messages/:messageId/poll/vote` (or `POST /api/conversations/:id/polls/:pollId/vote`)
* **Body:**
  ```json
  {
    "optionIds": ["opt-1"]
  }
  ```
* **Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Vote recorded successfully",
    "data": {
      "success": true,
      "conversationId": "conv-uuid",
      "messageId": "msg-uuid",
      "pollId": "poll-uuid",
      "poll": {
        "id": "poll-uuid",
        "question": "Which tech stack should we pick for the project?",
        "allowMultipleAnswers": false,
        "totalVotes": 1,
        "uniqueVotersCount": 1,
        "hasVoted": true,
        "userVotedOptionIds": ["opt-1"],
        "options": [
          {
            "id": "opt-1",
            "text": "Next.js + TypeScript",
            "order": 0,
            "voteCount": 1,
            "hasVoted": true,
            "voters": [
              { "userId": "user-uuid", "user": { "id": "user-uuid", "name": "Alice", "avatar": null }, "createdAt": "2026-11-04T21:50:00.000Z" }
            ]
          },
          {
            "id": "opt-2",
            "text": "Vite + React",
            "order": 1,
            "voteCount": 0,
            "hasVoted": false,
            "voters": []
          }
        ]
      }
    }
  }
  ```

---

## 4. Socket Error Codes

When a socket acknowledgement returns `{ "success": false }`, the `code` property indicates the cause:

| Error Code | Meaning | Recommended Frontend Action |
|---|---|---|
| `UNAUTHORIZED` | Token missing, invalid, or expired | Prompt re-login or refresh access token |
| `NOT_PARTICIPANT` | User is not a member of the conversation | Prevent access to this conversation |
| `RATE_LIMITED` | Message or typing limit exceeded | Display a temporary cooldown alert |
| `INVALID_PAYLOAD` | Content or payload validation failed | Check payload formatting and attachment URLs |
| `SERVER_ERROR` | Internal server exception | Retry with exponential backoff |
