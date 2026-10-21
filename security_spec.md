# Security Specification: TaskPulse Android & Firebase Task Management

## 1. System Overview & Data Invariants
TaskPulse is a real-time, offline-first task management application with synchronization between Android clients, web dashboards, and Firebase Firestore.

### Invariants:
1. **User Data Isolation**: A user can only access, create, update, or delete their own tasks, categories, and push notification tokens. Cross-user access is strictly denied.
2. **Relational Integrity**: Every task document must specify an immutable `userId` that matches the authenticated caller's UID (`request.auth.uid`).
3. **Input Sanitization & Boundary Limits**:
   - Task `title`: String, length between 1 and 250 characters.
   - Task `description`: String, length <= 2000 characters.
   - Task `priority`: Enum in `['low', 'medium', 'high', 'urgent']`.
   - Task `status`: Enum in `['todo', 'in_progress', 'completed', 'archived']`.
   - Task `dueDate`: Nullable string (ISO timestamp <= 64 chars).
   - Task `tags`: List of strings, maximum 10 elements, each string <= 32 chars.
   - `subtasks`: List of maps, maximum 20 elements.
   - `isCompleted`: Boolean.
   - `syncVersion`: Integer >= 1 for offline conflict resolution.
4. **Temporal Invariants**:
   - `createdAt` is immutable after creation.
   - `updatedAt` must be set to server timestamp `request.time` during mutation.
5. **Push Token Hardening**:
   - Notification tokens (`/users/{userId}/tokens/{tokenId}`) must belong to the authenticated user. Tokens cannot be read by anyone else.

---

## 2. The "Dirty Dozen" Adversarial Payloads
1. **Payload 1 (Identity Spoofing - Task Creation)**:
   `POST /tasks { id: "t1", userId: "victim_user_999", title: "Phishing Task" }` -> Denied (`userId != request.auth.uid`).
2. **Payload 2 (Ghost Field Injection)**:
   `POST /tasks { id: "t2", userId: "me", title: "Test", isAdmin: true, bypassSync: true }` -> Denied (`hasOnly` schema violation).
3. **Payload 3 (Denial of Wallet - Title Bomb)**:
   `POST /tasks { id: "t3", userId: "me", title: "A".repeat(50000) }` -> Denied (`title.size() <= 250`).
4. **Payload 4 (Tag Array Overflow)**:
   `POST /tasks { id: "t4", userId: "me", title: "Test", tags: [100 tags...] }` -> Denied (`tags.size() <= 10`).
5. **Payload 5 (ID Path Poisoning)**:
   `GET /tasks/../../admin/secret` -> Denied (`isValidId` regex & path bounds).
6. **Payload 6 (Timestamp Forgery)**:
   `POST /tasks { id: "t6", userId: "me", title: "Test", createdAt: timestamp(2099, 1, 1) }` -> Denied (`createdAt == request.time`).
7. **Payload 7 (Immutable Field Tampering)**:
   `UPDATE /tasks/t7 { userId: "other_user", createdAt: timestamp(2020, 1, 1) }` -> Denied (`userId == existing().userId && createdAt == existing().createdAt`).
8. **Payload 8 (Invalid Enum Value)**:
   `POST /tasks { id: "t8", userId: "me", title: "Test", priority: "critical_bypass" }` -> Denied (`priority in ['low', 'medium', 'high', 'urgent']`).
9. **Payload 9 (Unauthenticated Scrape / Blanket List)**:
   `GET /tasks (anonymous)` -> Denied (`request.auth != null && resource.data.userId == request.auth.uid`).
10. **Payload 10 (Push Notification Token Exfiltration)**:
    `GET /users/victim/tokens/tok1` by attacker -> Denied (User path mismatch).
11. **Payload 11 (Non-String Subtask Injection)**:
    `POST /tasks { id: "t11", userId: "me", title: "Test", subtasks: [12345, true] }` -> Denied (Type safety check on subtasks).
12. **Payload 12 (Cross-User Delete)**:
    `DELETE /tasks/victim_task` by attacker -> Denied (`existing().userId == request.auth.uid`).
