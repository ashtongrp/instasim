# Strict Architecture Constraints: Codespaces LLM Social Network

## 1. System Context
A remote-containerized (GitHub Codespaces), single-user Next.js application simulating a massive social network. The backend utilizes Next.js API routes to orchestrate *external* LLM generation, manage concurrency via an in-memory Mutex, and persist entity graphs to local JSON shards. The frontend is a strictly normalized, purely client-side rendered (CSR) view utilizing an LRU cache and heavily debounced state hydration to counter proxy network buffering.

## 2. Tech Stack
*   **Framework:** Next.js (App Router) strictly utilizing Client Components (`"use client"`) for primary views.
*   **LLM Provider:** External APIs only (OpenAI, Anthropic, Gemini). No local model hosting inside the container.
*   **State Management:** Zustand (normalized entity slices with custom LRU eviction).
*   **DOM Management:** `@tanstack/react-virtual` (mandatory for all lists > 20 items).
*   **Persistence:** Node.js `fs/promises` writing to `.data/*.json` files (must be `.gitignore`d).
*   **Styling:** Tailwind CSS.

## 3. Primary Objective
Construct a frictionless, infinitely expanding social media simulation. Generated content must be permanently cached to disk and efficiently rendered via a highly optimized, flat client-state graph that survives aggressive proxy buffering and container resource limits.

## 4. Prohibitions (Strict)
*   **NO NESTED STATE.** The client store will not hold posts inside user objects or comments inside post objects. All state must be flat entity maps.
*   **NO SSR FOR DATA.** Feeds and profile pages must mount as shells and fetch data client-side.
*   **NO LOCAL LLMS.** The container cannot support local model weights. API calls only.
*   **NO UNBUFFERED HYDRATION.** Zustand `upsert` actions from SSE streams must be debounced to prevent React re-render cascades when the Codespaces proxy flushes data chunks.
*   **NO UNVIRTUALIZED LISTS.** Any feed, search result, or comment section exceeding 20 items must use TanStack Virtual.
*   **NO RAM BLOAT.** Zustand stores must implement a strict 2,000-item LRU limit per entity slice.
*   **NO CONCURRENT WRITES.** All `fs.promises` write operations must be routed through a centralized Mutex queue.

## 5. Architecture Pattern
Feature-Sliced Design (FSD) enforcement.
*   `app/`: Routing and static shells.
*   `entities/`: Normalized state slices, models, interfaces, and LRU logic.
*   `features/`: Bound interactions (Search Bar, Like Button).
*   `widgets/`: Complex UI blocks (NewsFeed, ProfileSidebar).
*   `shared/`: UI primitives, Mutex utilities, ID generation, API clients, SSE hooks.

## 6. Strict TypeScript Contracts & Data Models
Prefix-based ID system is mandatory for entity tracking and debugging.

```typescript
// shared/types/ids.ts
export type UserId = `1${string}`;
export type PageId = `2${string}`;
export type GroupId = `3${string}`;
export type PostId = `4${string}`;
export type CommentId = `5${string}`;

export type EntityId = UserId | PageId | GroupId | PostId | CommentId;

// entities/user/model.ts
export interface User {
  id: UserId;
  name: string;
  handle: string;
  avatarUrl: string;
  bio: string;
  groupIds: GroupId[];
  lastAccessed: number; // Required for LRU eviction
}

// entities/post/model.ts
export interface Post {
  id: PostId;
  authorId: UserId | PageId;
  content: string;
  imageUrl?: string;
  likesCount: number;
  commentIds: CommentId[];
  timestamp: number;
  lastAccessed: number; // Required for LRU eviction
}

// shared/types/state.ts
export interface NormalizedState {
  users: Record<UserId, User>;
  posts: Record<PostId, Post>;
  // ... other entities
}
```

## 7. State Management & LRU Eviction
Zustand will hold the `NormalizedState`. An internal mechanism must sweep and delete the oldest entries when the key count exceeds `MAX_ENTITIES` (2000).

```typescript
// entities/post/store.ts
import { create } from 'zustand';
import { PostId, Post } from '@/shared/types';

const MAX_POSTS = 2000;

interface PostStore {
  posts: Record<PostId, Post>;
  upsertPosts: (newPosts: Post[]) => void;
  evictStale: () => void;
}

export const usePostStore = create<PostStore>((set, get) => ({
  posts: {},
  upsertPosts: (newPosts) => set((state) => {
    const nextPosts = { ...state.posts };
    const now = Date.now();
    newPosts.forEach(post => { 
      nextPosts[post.id] = { ...post, lastAccessed: now }; 
    });
    return { posts: nextPosts };
  }),
  evictStale: () => set((state) => {
    const keys = Object.keys(state.posts) as PostId[];
    if (keys.length <= MAX_POSTS) return state;
    
    // Sort by lastAccessed ascending and drop the oldest
    const sorted = keys.sort((a, b) => state.posts[a].lastAccessed - state.posts[b].lastAccessed);
    const toKeep = sorted.slice(-MAX_POSTS);
    
    const nextPosts: Record<PostId, Post> = {} as Record<PostId, Post>;
    toKeep.forEach(id => { nextPosts[id] = state.posts[id]; });
    return { posts: nextPosts };
  })
}));
```

## 8. Data Flow, Concurrency & Proxy Resilience
1.  **Trigger:** Client POSTs to `/api/generate`.
2.  **Server Queue:** The API route acquires a Mutex lock for the target JSON files.
3.  **Generation & Streaming:** Server triggers external LLM API, assigns IDs, writes to disk (holding the lock), and streams SSE to the client. Lock is released.
4.  **Debounced Hydration:** Client receives SSE. To combat GitHub's port-forwarding proxy dumping chunks all at once, client buffers incoming JSON and commits to Zustand in debounced batches (e.g., every 100ms).
5.  **Fallback & Reconnect:** Client SSE listener implements exponential backoff (1s, 2s, 4s, 8s) for disconnects. On reconnect, it seamlessly falls back to polling `/api/sync?since=<timestamp>` to fetch missed disk writes.

## 9. Component Boundaries
Components do not fetch relational data. They are passed an ID and select their data from the store.

*   `<Feed />`: Virtualizes a list of `PostId`s. Renders `<PostItem postId={id} />`.
*   `<PostItem />`: Accepts `postId`. Selects `Post`. Renders `<Avatar userId={post.authorId} />`. 
*   `<Avatar />`: Accepts `userId`. Selects `User` to render image.
