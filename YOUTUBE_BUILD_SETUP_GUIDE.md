# YouTube Build & Setup Guide: Coding with AI (Vercel Eve, Clerk Billing, & Convex)

This document is a comprehensive, step-by-step setup and configuration guide based directly on the video tutorial **"Let's Build a Career Guide AI Agent with Fable 5 | Beginner Series Ep #13 (Vercel Eve, Billing)"** by Sonny Sangha (PAPAFAM).

Use this document as your reference checklist whenever you want to bootstrap a brand-new Next.js application using an **AI orchestration architecture** with Clerk (auth & dual-sided billing), Convex (reactive database with vector search), and Vercel Eve (AI Agent).

---

## 📅 Video Timestamps & Progression Outline

* **00:00 - 14:20** App Demo, Agent Folder Structure, and Architecture Overview
* **14:21 - 21:20** Clerk Billing (B2C & B2B) Setup
* **21:21 - 37:41** Phase 1: Initialize Next.js & Clerk CLI
* **37:42 - 45:23** Phase 2: Running Locally & Connecting Git Worktrees
* **45:24 - 58:11** Phase 3: Install & Scaffold Vercel Eve + AI Gateway Key Setup
* **58:12 - 01:16:19** Phase 4: Custom Clerk-Authenticated Eve Channels & Next.js Mount
* **01:16:20 - 01:36:19** Phase 5: Setting Up Convex Reactive Backend & Vector Search

---

## 🛠️ Phase 1: Next.js 16 & Clerk Initialization

### 1. Create the Next.js App
Initialize a new Next.js project with Tailwind CSS and TypeScript:
```bash
npx create-next-app@latest my-agentic-app \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*"
```

### 2. Clerk Signup & CLI Setup
To avoid manually copy-pasting API keys from the browser, Sonny sets up and uses the Clerk CLI to authenticate directly in the terminal:
```bash
# Install Clerk CLI globally or run it via npx
npx clerk login
```

### 3. Initialize Clerk inside your Next.js App
Authenticate your codebase with your Clerk project using:
```bash
npx clerk init
```
This automatically updates your local `.env.local` file with:
```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
```

---

## 💳 Billing Configuration: B2C vs. B2B Plans

In the video, Sonny reviews the dual-sided Billing strategy using Clerk's pricing dashboard:

1. **User Billing (B2C):** Set up a **Personal Pro Plan** inside Clerk.
2. **Organization Billing (B2B):** Set up a **Company Pro Plan** billed to organizations.
3. Configure your slugs in your code (`PRO_PLAN = "pro"` and `COMPANY_PRO_PLAN = "company-pro"`).

These slugs are evaluated in the server-side billing middleware and hooks (detailed in Phase 4).

---

## 🤖 Phase 2: Installing and Scaffolding Vercel Eve

Vercel Eve treats AI Agents as a modular directory. You compile and run them inside your Next.js workspace.

### 1. Install Vercel Eve & Concurrently
Install the required agent packages and a concurrency helper to run your app and agent together:
```bash
npm install eve
npm install -D concurrently
```

### 2. Scaffold the Starter Eve Agent
Generate the agent files and a default template:
```bash
npx eve init
```
This scaffolds the `agent/` folder containing:
- `agent/agent.ts` (Model configuration)
- `agent/instructions.md` (System prompt instructions)
- `agent/tools/` (Agent capabilities/functions)
- `agent/channels/` (Agent inbound entrypoints)

### 3. Vercel AI Gateway Setup
To use Anthropic models (like Claude Sonnet 5) with caching and rate-limiting, route requests through Vercel's AI Gateway. Configure your environment variable in `.env.local`:
```env
# Point Eve to your AI Gateway endpoint
AI_GATEWAY_API_KEY=gw_...
```

Configure your model definition inside `agent/agent.ts`:
```typescript
// agent/agent.ts
import { defineAgent } from "eve";

export default defineAgent({
  model: "anthropic/claude-sonnet-5",
  reasoning: "medium", // Enables moderate internal reflection before responding
});
```

### 4. Run the Agent Locally for Testing
Run the Eve interactive CLI dev console:
```bash
# Run agent in development mode
npx eve dev
```
You can now speak directly to your agent inside the terminal and test standard tool calls!

---

## 🔐 Phase 3: Wiring Clerk Auth into the Vercel Eve Agent

To lock agent access behind your paid personal subscriptions, you must replace the scaffolded placeholder authentication with the secure Clerk authenticator channel.

### 1. Configure the Clerk-Authenticated Eve Channel
Update the Eve channel middleware to inspect Clerk JWT credentials and enforce premium gates server-side:

```typescript
// agent/channels/eve.ts
import { eveChannel } from "eve/channels/eve";
import { ForbiddenError } from "eve/channels/auth";
import { createClerkClient } from "@clerk/backend";
import { userSubscriptionIsPro } from "@/lib/billing";

function clerkAuth() {
  return async (request: Request) => {
    const clerk = createClerkClient({
      secretKey: process.env.CLERK_SECRET_KEY!,
      publishableKey: process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY!,
    });

    const requestState = await clerk.authenticateRequest(request);
    const auth = requestState.toAuth();

    // Not authenticated → reject
    if (!auth || !auth.userId) return null;

    // Billing check gate (User subscription must be active Pro)
    const isPro = await userSubscriptionIsPro(clerk, auth.userId);
    if (!isPro) {
      throw new ForbiddenError({
        message: "Please upgrade to Personal Pro to use the AI Career Agent.",
      });
    }

    // Forward the authenticated userId as the principalId to tools
    return {
      authenticator: "app",
      principalId: auth.userId,
      principalType: "user",
      attributes: {},
    };
  };
}

export default eveChannel({
  auth: [clerkAuth()],
});
```

### 2. Mount Vercel Eve into Next.js Route Endpoints
Wrap your Next.js application configuration to automatically mount the `/eve/v1/*` endpoint proxy:

```typescript
// next.config.ts
import type { NextConfig } from "next";
import { withEve } from "eve/next";

const nextConfig: NextConfig = {
  reactCompiler: true,
};

export default withEve(nextConfig);
```

### 3. Create the Next.js Custom Proxy Middleware
Next.js utilizes `proxy.ts` at the root of the project. Protect your application routes while keeping your public landing page, Clerk widgets, and Eve channel endpoint accessible:

```typescript
// proxy.ts
import { clerkMiddleware, createRouteMatcher } from "@clerk/nextjs/server";

const isPublicRoute = createRouteMatcher([
  "/",
  "/sign-in(.*)",
  "/sign-up(.*)",
  "/eve(.*)", // Let the Eve authenticated channel handle its own auth/billing
  "/api(.*)",
]);

export default clerkMiddleware(async (auth, req) => {
  if (!isPublicRoute(req)) {
    await auth.protect();
  }
});

export const config = {
  matcher: [
    "/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|webmanifest)).*)",
    "/(api|trpc)(.*)",
  ],
};
```

---

## 🗄️ Phase 4: Adding Convex as the Backend & Vector Search

Convex handles real-time synchronization, file storage, and similarity-based vector searches (using text embeddings) for job matches.

### 1. Install Convex & Setup Workspace
Install Convex and configure the workspace variables:
```bash
npm install convex
# Link your workspace project on Convex cloud
npx convex dev
```
This creates a new Convex deployment and establishes a `.env.local` hook:
```env
NEXT_PUBLIC_CONVEX_URL=https://...
```

### 2. Configure Database Schema with Vector Search Indexes
Define your tables and vector indexes inside `convex/schema.ts`. Sonny leverages OpenAI's `text-embedding-3-small` (dimension 1536) for semantic vector comparison:

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  users: defineTable({
    clerkId: v.string(),
    name: v.string(),
    email: v.string(),
    username: v.string(),
  }).index("by_clerkId", ["clerkId"]),

  jobs: defineTable({
    title: v.string(),
    description: v.string(),
    skillsRequired: v.array(v.string()),
    embedding: v.optional(v.array(v.float64())), // Vector embedding
  }).vectorIndex("by_embedding", {
    vectorField: "embedding",
    dimensions: 1536,
  }),
});
```

### 3. Implement the Semantic Search Action
Write a server-side action on Convex to perform vector calculations. In the video, Sonny shows how to turn search text into an embedding via the Vercel AI Gateway (or OpenAI API) and query the reactive vector index:

```typescript
// convex/embeddings.ts
import { action } from "./_generated/server";
import { v } from "convex/values";

export const getVectorMatches = action({
  args: {
    queryText: v.string(),
  },
  handler: async (ctx, args) => {
    // 1. Get embedding for search query via AI Gateway
    const response = await fetch("https://gateway.ai.cloudflare.com/v1/.../openai/embeddings", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${process.env.OPENAI_API_KEY}`,
      },
      body: JSON.stringify({
        input: args.queryText,
        model: "text-embedding-3-small",
      }),
    });

    const resData = await response.json();
    const queryVector = resData.data[0].embedding;

    // 2. Query Convex Vector Index
    const matches = await ctx.vectorSearch("jobs", "by_embedding", {
      vector: queryVector,
      limit: 5,
    });

    return matches;
  },
});
```

---

## 🏃‍♂️ Running the Combined Local Workspace

To simplify your development experience, Sonny recommends modifying the `scripts` object inside `package.json` to run your entire stack concurrently using a single command:

```json
"scripts": {
  "dev": "next dev",
  "dev:all": "concurrently -n next,convex -c blue,magenta \"pnpm dev\" \"pnpm convex\"",
  "convex": "convex dev",
  "agent": "eve dev"
}
```

Now, simply execute:
```bash
npm run dev:all
```
This runs your Next.js application frontend, synchronization backend, and the interactive Vercel Eve Agent! Use this template to launch similar next-gen AI platforms from scratch.