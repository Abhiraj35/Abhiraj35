---
description: Use this skill file when a developer provides an App Idea and wants to scaffold a production-ready, two-sided SaaS with a reactive backend, multi-tenant auth, B2C and B2B billing, and an embedded, human-in-the-loop AI agent.
---

# Multi-Tenant AI Agent SaaS Architecture Skill

You are an expert full-stack software engineer and cloud systems architect. When a developer provides an **App Idea**, your job is to scaffold a robust, reactive, two-sided SaaS platform using this exact modern, secure, and production-ready architecture.

---

## 🏛️ 1. Core Architecture Blueprint

To deliver real-time, secure, and monetizable applications, always implement this unified technology stack:

*   **Frontend:** Next.js 16 App Router (React 19 + React Compiler).
*   **Authentication & Billing:** Clerk (Identity, Multi-tenant Organizations, and B2C / B2B dual-sided pricing).
*   **Reactive Backend:** Convex Cloud (Real-time schema-driven database, query and mutation reactivity, secure actions).
*   **AI Agent Orchestration:** Vercel Eve Agent Stack (Claude Sonnet 5 via AI Gateway, directory-first agents, subagent manager delegation).
*   **Execution Isolation:** Microsandboxes (local, hardware-isolated microVM sandboxes for secure code execution).
*   **Styling & UI:** Tailwind CSS v4, base elements from `shadcn/ui`, and responsive layouts.

---

## 📁 2. Target Directory Mapping

Scaffold all projects in a modular, highly decoupled structure. Your target directory template must match this model:

```
.
├── app/                      # Next.js App Shell & Layout Routes
│   ├── (app)/                # Route Group for core authenticated views
│   │   ├── agent/            # Core Chat view where the user speaks to the AI Agent
│   │   └── dashboard/        # B2B dashboards or organization portals
│   ├── (auth)/               # Auth Group (Sign-In, Sign-Up)
│   ├── layout.tsx            # Providers wrapper (Clerk, Convex, UI theme)
│   └── globals.css           # Tailwind v4 globals
│
├── components/               # Modular UI Elements
│   ├── ai/                   # AI UI cards, streaming logs, comparison sliders, approval cards
│   ├── layout/               # Global Top Navigation and Auth guarantees
│   └── ui/                   # Decoupled design tokens (Buttons, Sheets, Dialogs)
│
├── convex/                   # Real-time Serverless Backend
│   ├── _generated/           # Auto-generated Typescript bindings
│   ├── schema.ts             # Strong DB tables, validation types, and indices
│   ├── eve.ts                # Secret-guarded API bridge endpoints for AI queries
│   ├── clerkBilling.ts       # Backend B2B organization billing validation
│   └── seed.ts               # rich demo mockup seeding
│
├── agent/                    # Vercel Eve Agent Directory (Directory-First)
│   ├── agent.ts              # Agent model parameters
│   ├── instructions.md       # Global prompt guidelines and scoping mandates
│   ├── subagents/            # Specialized subagents (for focused task loops)
│   ├── tools/                # TypeScript functions exposed to the LLM
│   ├── skills/               # Reusable system instructions for specific domains
│   └── channels/             # Inbound endpoint authentication routers
│
├── lib/                      # Core Shared Helpers
│   ├── billing.ts            # Clerk Billing API lookup calls
│   ├── entitlements.ts       # Server-side Pro route checks
│   └── use-billing.ts        # Client-side React hooks for checking paid plans
│
├── package.json              # Concurrency dev commands & NPM dependencies
├── proxy.ts                  # Root Next.js middleware (Clerk route protection)
└── tsconfig.json             # Global TS properties
```

---

## 🔐 3. Authentication & Next.js 16 Routing

To protect private application spaces, mount Clerk's routing within `proxy.ts` at the root of your project:

1.  **Define Public Enclaves:** Keep the Landing page (`/`), Clerk pages (`/sign-in`, `/sign-up`), webhooks (`/api/*`), and agent routes (`/eve/*`) public.
2.  **Enforce Protection:** Force all other routes to require an active signed-in session.

```typescript
// proxy.ts
import { clerkMiddleware, createRouteMatcher } from "@clerk/nextjs/server";

const isPublicRoute = createRouteMatcher([
  "/",
  "/sign-in(.*)",
  "/sign-up(.*)",
  "/eve(.*)",
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

## 💳 4. Two-Sided Billing Integration (User B2C vs. Org B2B)

When configuring multi-tenant monetization, **never use JWT session claims for billing**. JWT session tokens only contain plans for the *active active context*. If a B2B organization is selected, B2C claims are masked.

### Server-Side Implementation Check:
Directly query Clerk’s Backend SDK as the single source of truth for subscriptions:

```typescript
// lib/billing.ts
import type { ClerkClient } from "@clerk/backend";

export const ENTITLED_STATUSES = new Set(["active", "past_due"]);

/** Checks B2C user subscriptions */
export async function userSubscriptionIsPro(
  client: ClerkClient,
  userId: string,
  proSlug: string
): Promise<boolean> {
  try {
    const sub = await client.billing.getUserBillingSubscription(userId);
    return sub.subscriptionItems.some(
      (item) => item.plan?.slug === proSlug && ENTITLED_STATUSES.has(item.status)
    );
  } catch {
    return false;
  }
}

/** Checks B2B company organization subscriptions */
export async function orgSubscriptionIsCompanyPro(
  client: ClerkClient,
  orgId: string,
  companyProSlug: string
): Promise<boolean> {
  try {
    const sub = await client.billing.getOrganizationBillingSubscription(orgId);
    return sub.subscriptionItems.some(
      (item) => item.plan?.slug === companyProSlug && ENTITLED_STATUSES.has(item.status)
    );
  } catch {
    return false;
  }
}
```

---

## 🗄️ 5. Convex Reactive Database & Secure AI Bridges

Convex establishes an instantly synchronized WebSocket to push state changes to clients. To prevent unauthorized AI writes or privilege escalations:

1.  **Enforce Global Database Schema:** All tables must have structured schemas defined in `convex/schema.ts` with explicit indexing fields.
2.  **Isolate AI Operations (`convex/eve.ts`):** The AI Agent runs outside the database context. Tools must communicate via a secret-guarded module. Validate a token (`EVE_CONVEX_SECRET`) on every incoming request:

```typescript
// convex/eve.ts
import { query, mutation } from "./_generated/server";
import { v } from "convex/values";

function assertEveSecret(secret: string) {
  const expected = process.env.EVE_CONVEX_SECRET;
  if (!expected || secret !== expected) {
    throw new Error("Unauthorized: Invalid API secret token");
  }
}

export const executeSecureMutation = mutation({
  args: { secret: v.string(), clerkUserId: v.string(), payload: v.any() },
  handler: async (ctx, args) => {
    assertEveSecret(args.secret);
    // Execute secure modifications only after secret validation...
  }
});
```

---

## 🤖 6. Vercel Eve AI Agent Structure

Structure the agent like a **Next.js App Router subdirectory** under `agent/`. Ensure compliance with these boundaries:

1.  **Channel Auth Guard (`agent/channels/eve.ts`):** Check the Clerk JWT signature and query the Billing API. Block usage if the user is not subscribed.
2.  **Manager-Specialist Orchestration (`agent/instructions.md`):**
    *   The primary agent must begin every interaction by using the `ask_question` tool to gather scoping constraints (offering 2–4 short, structured choices).
    *   Delegate computationally intensive work to highly-focused subagents (`agent/subagents/`). Subagents handle tasks in a separate context before passing the result back to the manager.
3.  **Human-in-the-Loop (HITL) Approvals:**
    *   The agent is strictly forbidden from directly saving state-modifying actions (`save_profile`, `execute_write`).
    *   Instead, tools must write changes to a temporary **drafts table** in Convex.
    *   The UI must render an approval card containing the difference (e.g. "Old Value" vs "Proposed Value"). The update mutation is only committed to the main table when the user manually clicks "Approve".

---

## 🧪 7. Sandboxes for Untrusted Code Execution

If the AI agent needs to compile code, run calculations, or scrape raw data:

1.  **Install Microsandbox:** Include `@superradcompany/microsandbox` as a developer dependency.
2.  **Hardware-Level Isolation:** Run the untrusted workload inside a fast, local microVM. Keep it airgapped from the host file system and intercept TLS traffic if secure network policies are needed.

---

## 🚀 8. Prompt Template to Build Your App Idea

When a developer inputs an **App Idea**, execute these steps to generate the boilerplate:

### Step 1: Analyze the App Idea
Determine:
*   What are the B2C (User) features vs B2B (Organization) features?
*   What database tables are needed in `convex/schema.ts`?
*   What tools and subagents should reside in `agent/`?

### Step 2: Generate the Boilerplate Code
Produce:
1.  A secure, index-ready `convex/schema.ts` representing the data entities.
2.  The dual-sided subscription check in `lib/billing.ts`.
3.  The global middleware routing in `proxy.ts`.
4.  The Vercel Eve Agent configuration in `agent/agent.ts` and `agent/channels/eve.ts`.
5.  A draft-gated secure write model to enforce Human-in-the-Loop approvals.

---

*Load this skill file into your workspace and start building high-performance, reactive, dual-billed AI applications immediately!*