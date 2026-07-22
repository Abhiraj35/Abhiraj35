# Architecture & Infrastructure Guide: Next-Gen Career Platform (CareerConnect)

Welcome! This guide is written especially for developers (from beginners to advanced) who want to understand **how this production-grade, two-sided SaaS is engineered**.

Rather than focusing on career counseling or job descriptions, this document focuses strictly on the **systems architecture, developer patterns, directory structure, security bridges, dual-sided billing, AI agent orchestrations, and sandboxes** that make up this modern platform. You can use these identical blueprints and architectural patterns to launch any modern web application or SaaS from scratch.

---

## 🗺️ 1. High-Level System Architecture

CareerConnect is designed around a modern, event-driven, and highly reactive systems architecture:

```
┌────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND                                  │
│                 Next.js 16 App Router (React 19)                       │
│      (Clerk Provider, Convex React Hooks, Custom AI Chat Components)   │
└──────────────────┬───────────────────┬───────────────────┬─────────────┘
                   │                   │                   │
             (HTTPS / Auth)      (Reactive Sync)     (Secure HTTPS)
                   │                   │                   │
                   ▼                   ▼                   ▼
┌──────────────────────┐  ┌──────────────────────┐  ┌────────────────────┐
│      AUTH & BILLING  │  │   REACTIVE BACKEND   │  │   AI AGENT CHIPS   │
│        Clerk SaaS    │  │     Convex Cloud     │  │  Vercel Eve Agent  │
│                      │  │                      │  │  (Claude Sonnet 5) │
│ • Multi-tenant Orgs  │  │ • Real-time DB Sync  │  │                    │
│ • B2C subscription   │◀─┼─• Secure mutations   │◀─┼─• Specialist       │
│ • B2B org-billing    │  │ • Embeddings Vector  │  │   subagents        │
│ • Secure Webhooks    │  │   Indices (Search)   │  │ • Human-in-the-    │
│                      │  │                      │  │   Loop Approvals   │
└──────────────────────┘  └──────────────────────┘  └────────────────────┘
```

### The Core Technology Quadrumvirate

1. **Next.js 16 (App Router + React Compiler):** Serves as the interactive shell, delivering server-side rendering (SSR), optimized UI components via `shadcn/ui`, and native client-side routing. It mounts the AI runtime inside its own endpoints (`/eve/*`) through middleware proxies.
2. **Clerk (Auth + Orgs + Billing):** Standard Auth platforms only verify identity. Clerk handles **identity, multi-tenant B2B organizations, and integrated subscription billing (both personal-tier and business-tier)** without having to manually code Stripe webhooks.
3. **Convex (Reactive Cloud Backend):** Unlike traditional SQL/NoSQL databases that require REST/GraphQL polling, Convex is fully **reactive**. If an applicant's stage changes, or a new social feed post is created, Convex instantly pushes the updated state to all connected UI clients using automatic WebSocket sync.
4. **Vercel Eve (Agent Stack):** A cutting-edge, file-system-based TypeScript agent framework. Rather than a messy single-prompt LLM wrapper, Eve treats the AI Agent as a **structured directory** containing subagents, isolated tools, and system instructions. It runs as a durable workflow that checkpoints and survives runtime crashes.

---

## 📁 2. Directory Structure & File Architecture

The repository uses a predictable, modular workspace structure. Here is how the folders are organized and why:

```
.
├── app/                      # Next.js 16 Application Shell & Pages
│   ├── (app)/                # Route Group for Core App (Feed, Jobs, Profile)
│   │   ├── agent/            # Core Chat view where the user speaks to "Eve"
│   │   └── company/          # Hiring panel, job manager, & applicant pipeline
│   ├── (auth)/               # Authentication Route Group (Sign-In, Sign-Up)
│   ├── layout.tsx            # Global providers (Clerk, Convex, Themes, Toast)
│   └── globals.css           # Global Tailwind CSS v4 styling rules
│
├── components/               # Highly modular, reusable UI components
│   ├── ai/                   # AI UI components: Agent chat, Markdown, approval gates
│   ├── layout/               # Global Top Nav bar, ensures database sync checks
│   └── ui/                   # Decoupled Tailwind design elements (Button, Dialog)
│
├── convex/                   # Reactive backend database schema & logic
│   ├── _generated/           # Auto-generated Typescript types based on schemas
│   ├── schema.ts             # Decoupled database tables, models, & indexes
│   ├── eve.ts                # Trusted API endpoints called exclusively by AI Agent
│   ├── clerkBilling.ts       # Secure billing verification routines
│   └── seed.ts               # Database seeding routine with rich mockup data
│
├── agent/                    # Vercel Eve Agent Structure (Treats AI as a Directory)
│   ├── agent.ts              # Agent model definition & parameters
│   ├── instructions.md       # Global coaching instructions and constraints
│   ├── subagents/            # Specialist subagents (Shortlists, Outreaches, Plans)
│   ├── tools/                # Hard-coded TypeScript functions the AI is allowed to run
│   ├── skills/               # Core system prompt extensions & behaviors
│   └── channels/             # Routing & authentication for inbound HTTP triggers
│
├── lib/                      # Core utility functions & shared logic
│   ├── billing.ts            # Server-side Billing verification APIs
│   ├── entitlements.ts       # Route guard assertions (isPro, isCompanyPro)
│   └── use-billing.ts        # Client-side React hooks for checking paid entitlements
│
├── package.json              # NPM dependencies & workspace scripts
├── proxy.ts                  # Next.js 16 Proxy Middleware (Clerk auth protection)
└── tsconfig.json             # Root TypeScript configuration
```

---

## 🔐 3. Authentication & Next.js 16 Routing: The `proxy.ts` Middleware

In Next.js 16, standard middleware conventions are managed by `proxy.ts` at the repository root. This file intercepts every inbound browser request to guard private pages or mount routes.

```typescript
// proxy.ts
import { clerkMiddleware, createRouteMatcher } from "@clerk/nextjs/server";

// Define which routes do not require login
const isPublicRoute = createRouteMatcher([
  "/",               // Landing Page
  "/sign-in(.*)",    // Clerk Auth
  "/sign-up(.*)",    // Clerk Auth
  "/eve(.*)",        // Mount endpoint for Vercel Eve AI Agent channel
  "/api(.*)",        // Clerk webhooks or backend webhooks
]);

export default clerkMiddleware(async (auth, req) => {
  if (!isPublicRoute(req)) {
    await auth.protect(); // Redirects signed-out users to Clerk Sign-In
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

## 💳 4. Two-Sided Subscriptions & B2C/B2B Clerk Billing

A primary challenge in SaaS architecture is supporting a dual-billing model:
* **B2C User Subscriptions (Personal Pro):** Billed to an individual's personal account to unlock the AI Career Agent.
* **B2B Organization Subscriptions (Company Pro):** Billed to a business unit (Clerk Organization) to unlock premium applicant pipelines and skill insights for all organization members.

### Why session tokens fall short
Standard architectures rely on JWT session claims to check billing tiers. However, session tokens only carry claims for the **currently active context**. If a user is active in their organization context, their personal B2C Pro plan claims are hidden!

### The Solution: Direct Server-Side API Lookups
To resolve this, CareerConnect bypasses token claims and queries the **Clerk Billing API** as the absolute source of truth.

```typescript
// lib/billing.ts
import type { ClerkClient } from "@clerk/backend";
import { PRO_PLAN, COMPANY_PRO_PLAN } from "./ai-features";

export const ENTITLED_STATUSES = new Set(["active", "past_due"]);

/** Checks if a human is on the personal B2C Pro plan */
export async function userSubscriptionIsPro(
  client: ClerkClient,
  userId: string,
): Promise<boolean> {
  try {
    const sub = await client.billing.getUserBillingSubscription(userId);
    return sub.subscriptionItems.some(
      (item) => item.plan?.slug === PRO_PLAN && ENTITLED_STATUSES.has(item.status),
    );
  } catch {
    return false; // Fails closed on API errors
  }
}

/** Checks if a company (Clerk Organization) is on the B2B Pro plan */
export async function orgSubscriptionIsCompanyPro(
  client: ClerkClient,
  orgId: string,
): Promise<boolean> {
  try {
    const sub = await client.billing.getOrganizationBillingSubscription(orgId);
    return sub.subscriptionItems.some(
      (item) => item.plan?.slug === COMPANY_PRO_PLAN && ENTITLED_STATUSES.has(item.status),
    );
  } catch {
    return false;
  }
}
```

On the frontend, Clerk's experimental client-side hooks are used to prevent flashes of unauthorized UI layout:
```typescript
// lib/use-billing.ts
import { useSubscription } from "@clerk/nextjs/experimental";
import { PRO_PLAN, COMPANY_PRO_PLAN } from "@/lib/ai-features";

export function usePersonalPro() {
  const { data, isLoading } = useSubscription({ for: "user" });
  const isPro = data?.subscriptionItems.some(item => item.plan.slug === PRO_PLAN && item.status === "active") ?? false;
  return { isPro, isLoaded: !isLoading };
}
```

---

## 🗄️ 5. Convex Reactive Database & Secure Agent Bridges

Traditional agent frameworks give LLMs read/write tokens that query databases directly. This is unsafe; an LLM could overwrite critical records or access other tenants' data.

To prevent this, CareerConnect implements a **Secret-Guarded AI Bridge** inside Convex.

```
┌────────────────┐                     ┌──────────────────┐                     ┌─────────────────┐
│  AI Runtime    ├────────────────────▶│ Convex Mutation  ├────────────────────▶│ Database Row    │
│  (Vercel Eve)  │  With: EVE_SECRET  │ (eve.ts Query)   │  Validate & Save   │ (profiles/jobs) │
└────────────────┘                     └──────────────────┘                     └─────────────────┘
```

1. **Trusted Runtime Session:** When the Eve Agent makes a tool call, it grabs the active Clerk user ID from the authenticated request channel (`agent/channels/eve.ts`).
2. **One-Shot API Query:** The tool instantiates a `ConvexHttpClient` and calls the `api.eve.*` module. It passes along an encrypted environment secret (`EVE_CONVEX_SECRET`) and the user's ID.
3. **Internal Auth Gate:** Convex verifies the secret on the server before executing any query or mutation:

```typescript
// convex/eve.ts
import { query, mutation, QueryCtx, MutationCtx } from "./_generated/server";
import { v } from "convex/values";

function assertEveSecret(secret: string) {
  const expected = process.env.EVE_CONVEX_SECRET;
  if (!expected || secret !== expected) {
    throw new Error("Unauthorized: invalid Eve secret");
  }
}

/**
 * Enforces security by checking the system token before returning
 * the profile info for a specific clerkUserId.
 */
export const getUserContext = query({
  args: { secret: v.string(), clerkUserId: v.string() },
  handler: async (ctx, args) => {
    assertEveSecret(args.secret); // Secret validation

    // Fetch the target user's row safely
    const user = await ctx.db
      .query("users")
      .withIndex("by_clerkId", (q) => q.eq("clerkId", args.clerkUserId))
      .unique();
    if (!user) return null;

    // Build and return profile, experiences, and skills...
    const profile = await ctx.db
      .query("profiles")
      .withIndex("by_userId", (q) => q.eq("userId", user._id))
      .unique();

    return { userId: user._id, headline: profile?.headline ?? null, ... };
  },
});
```

---

## 🤖 6. Vercel Eve Agent Framework Setup

Vercel Eve models an agent like a **Next.js App Router directory**. Each file handles a specific agent responsibility:

### A. The Global Config (`agent/agent.ts`)
Defines the brain powering your agent. It utilizes Claude Sonnet 5 routed through Vercel's AI Gateway:
```typescript
import { defineAgent } from "eve";

export default defineAgent({
  model: "anthropic/claude-sonnet-5",
  reasoning: "medium", // Enables moderate internal reflection before responding
});
```

### B. The Auth Endpoint Channel (`agent/channels/eve.ts`)
Mounts the agent under `/eve` inside the Next.js application, enforcing Clerk server-side auth and entitlement gates:
```typescript
import { eveChannel } from "eve/channels/eve";
import { createClerkClient } from "@clerk/backend";
import { userSubscriptionIsPro } from "@/lib/billing";

function clerkAuth() {
  return async (request: Request) => {
    const clerk = createClerkClient({ secretKey: process.env.CLERK_SECRET_KEY! });
    const authState = await clerk.authenticateRequest(request);
    const auth = authState.toAuth();

    if (!auth?.userId) return null; // Falls back to local login or 401

    // Pro Subscription Gate (enforces paid access before executing agent cycles!)
    const isPro = await userSubscriptionIsPro(clerk, auth.userId);
    if (!isPro) {
       throw new Error("Upgrade to Pro to use the AI Career Agent.");
    }

    return { authenticator: "app", principalId: auth.userId, principalType: "user" };
  };
}

export default eveChannel({ auth: [clerkAuth()] });
```

### C. Specialist Subagent Delegation
The agent is instructed to behave like a manager: it coordinates tasks, but delegates intensive workloads to isolated specialist subagents (`agent/subagents/*`).

* `job-scout`: Handles read-only job listings scoring.
* `profile-writer`: Designs drafts of user profiles.
* `outreach-writer`: Compiles outreach email and message bodies.
* `career-planner`: Sequences a structured 30/60/90-day plan.

When delegation is triggered, Eve fires a short descriptive line to the UI, spinning up a **Subagent Live Processing Card** while delegating the cycle inside a secure context.

---

## 🛑 7. Sandboxes & Human-in-the-Loop Approvals

Executing untrusted LLM-generated code or persisting data immediately poses major risks. CareerConnect solves this on two fronts:

### MicroVM Workload Isolation (The Sandbox)
If your agent needs to write and run custom code (e.g., executing Python script to calculate math, parsers, scrapers), it uses **Microsandbox** (powered by `@superradcompany/microsandbox` in `package.json`).
* It spins up lightweight, local hardware-isolated microVMs in **under 100 milliseconds**.
* The VM is completely airgapped, running code in a secure container away from your server's host file system.

### Human-in-the-Loop Approval Gating (The UI Bridge)
For database edits, Eve is prohibited from saving anything without a human's confirmation. This is handled using structured drafts and a dual-button confirmation card.

```
┌─────────────────┐       Draft persistent       ┌─────────────────┐
│ Specialist LLM  ├─────────────────────────────▶│  Convex Drafts  │
└─────────────────┘                              └────────┬────────┘
                                                          │
                                                Pushes draft to React UI
                                                          │
                                                          ▼
                                                 ┌─────────────────┐
                                                 │   Approval UI   │
                                                 │ [Approve]  [X]  │
                                                 └────────┬────────┘
                                                          │
                                              User clicks [Approve]
                                                          │
                                                          ▼
                                                 ┌─────────────────┐
                                                 │ Run saveTool()  │
                                                 │ (Writes to DB)  │
                                                 └─────────────────┘
```

1. **Draft Generation:** Instead of directly updating the `profiles` table, the subagent drafts the change and records it in a temporary `profileDrafts` database table.
2. **State Sync:** The draft ID is sent to the Frontend. The React client detects the pending draft and renders an interactive comparison UI (e.g. showing "Old Headline" vs "Proposed Headline").
3. **Execution Call:** Only when the user manually clicks "Approve", the component invokes the corresponding `save_profile_draft` Convex mutation, committing the proposal permanently to their profile.

---

## 🚀 8. How to Bootstrap This Architecture in Other Projects

You can use this exact architecture to start other SaaS systems. Here is your checklist for replication:

### Step 1: Initialize Workspace Packages
Organize your project with a workspace manager (such as `pnpm-workspace.yaml`):
```yaml
# pnpm-workspace.yaml
allowBuilds:
  esbuild: true
  sharp: true
```

### Step 2: Set Up Clerk Auth & Billing
1. Sign up on [Clerk](https://clerk.com), enable "Organizations" if building a multi-tenant B2B app, and configure your payment plans directly in the Clerk Billing dashboard.
2. Configure your project keys in `.env.local`:
   ```bash
   NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
   CLERK_SECRET_KEY=sk_test_...
   ```

### Step 3: Wire Convex Reactive Backend
1. Initialize convex inside your project:
   ```bash
   npm i convex
   npx convex dev
   ```
2. Build a clear schema in `convex/schema.ts` defining your application state tables.

### Step 4: Install and Mount Vercel Eve Agent
1. Install Eve packages:
   ```bash
   npm install eve
   ```
2. Wrap your `next.config.ts` configuration with Eve middleware:
   ```typescript
   import { withEve } from "eve/next";
   export default withEve({ reactCompiler: true });
   ```
3. Establish your agent structure inside `agent/` (add `agent.ts`, `instructions.md`, `tools/`, and `channels/`).
4. Run your concurrent dev engines:
   ```bash
   concurrently -n next,convex "pnpm next dev" "pnpm convex dev"
   ```

By aligning with this highly decoupled, secure, and reactive design, your application will scale easily to support thousands of active users, real-time sync, and robust AI integrations.