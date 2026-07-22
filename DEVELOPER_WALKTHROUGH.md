# Sequential Commit-by-Commit Walkthrough: CareerConnect

This developer walkthrough is designed to show you how this project was constructed step-by-step, from the initial setup to the finalized, polished product.

By following this sequential history, you can understand how to build similar full-stack AI-integrated projects. You will learn what files were added/modified at each step and the engineering rationale behind them.

---

## 🚀 The Development Timeline: A Chronological Reconstruction

```
  [ff6b4b5] Initial build
      │
  [9c14663] CareerConnect: Main stack, Clerk Auth, Convex schemas, Eve AI Agent, and basic components
      │
  [26bb962] Added B2B organization routing, company portals, job applicant pipelines, notifications
      │
  [ef64304] Fully linked seed data pipeline & Integrated Clerk Billing for Organizations (B2B Pro)
      │
  [6e3a0fc] Gated premium features with a unified B2C (User) & B2B (Company) billing layer
      │
  [ecd4ab4] Self-healed missing user database rows dynamically on authentication mutations
      │
  [eaeadad] Added AI job-match scoring UI, company page links, and streamlined navigation elements
      │
  [cef4d32] Implemented a Clerk-native multi-tenant onboarding gate for new companies
      │
  [0d04d72] Built real-time typeahead global search for jobs, companies, and people
      │
  [b025191] UI tweak: Lifted Clerk checkout drawers above the sticky top navigation
      │
  [372edde] Enhanced job results to highlight matching company profiles at the top
      │
  [041aeba] Created an applicant detail panel with full pipeline control states & cover letter reads
      │
  [9b9253f] Restructured Clerk Billing and expanded Vercel Eve capabilities with conversational chat
      │
  [046053d] Added toggles in the company panel to dynamically hide/show rejected applicants
      │
  [cb9ab51] Replaced modal applicant cards with a clean, fully-accessible dedicated detail page
      │
  [15ffb83] Implemented route-bouncing for authenticated users away from landing auth widgets
      │
  [8bc72c6] Reworked job posts as lazy-loading accordion lists with built-in action cards
      │
  [20aed99] Finalized Eve Agent instruction scoping, subagent handoffs, and Job Scout shortening
```

---

## 🛠️ Step-by-Step Milestones & Analysis

### Milestone 1: Core Setup & NextJS Integration
* **Commits:** `ff6b4b5` (Initial build) and `9c14663` (Build CareerConnect...)
* **Key Files:** `package.json`, `convex/schema.ts`, `proxy.ts`, `agent/agent.ts`, `agent/instructions.md`, `app/page.tsx`
* **What Happened:**
  - Placed Next.js 16 and React 19 in place.
  - Formulated the Convex database architecture by specifying core schemas inside `convex/schema.ts` (defining tables for `users`, `profiles`, `experiences`, `skills`, `savedJobs`, `jobs`, and `aiRuns`).
  - Integrated Clerk authentication via standard proxy middleware (`proxy.ts`).
  - Integrated **Vercel Eve** as a directory-first agent inside `agent/` with tool call definitions in `agent/tools/` for mapping agent queries back to Convex client query handlers securely.
* **Learning Takeaway:** When building an AI SaaS, always establish your database schema first, followed by auth routing, before attempting to build complex UI modules. This provides a structured schema for both the UI and the AI agent.

---

### Milestone 2: Transition to Two-Sided Market & Applicant Pipelines
* **Commit:** `26bb962` (Add company accounts...)
* **Key Files:** `convex/applications.ts`, `convex/companies.ts`, `app/(app)/company/page.tsx`
* **What Happened:**
  - Expanded the data model to support **Clerk Organizations** for company users.
  - Implemented multi-tenant CRUD actions for job postings, making sure queries are scoped to the authenticated organization `orgId`.
  - Added an application tracking table to manage candidates across custom stages: `submitted`, `reviewed`, `interviewing`, `offer`, `rejected`, and `withdrawn`.
  - Configured a real-time notification mechanism inside `convex/notifications.ts` to alert job seekers when their application stage changes.
* **Learning Takeaway:** Multi-tenancy must be enforced at the database query level (e.g., scoping reads via `.filter(q => q.eq(q.field("orgId"), activeOrgId))`). Never rely entirely on client-side state to gate data partitions.

---

### Milestone 3: Dual-Sided Subscriptions & Org Billing
* **Commits:** `ef64304` (Seed data and org-based billing...) and `6e3a0fc` (Gate org-only Pro features...)
* **Key Files:** `convex/clerkBilling.ts`, `lib/billing.ts`, `lib/entitlements.ts`
* **What Happened:**
  - Fully decoupled plan definitions for **B2C personal users** (`PRO_PLAN`) vs **B2B business units** (`COMPANY_PRO_PLAN`).
  - Designed the Clerk Billing integration inside `convex/clerkBilling.ts`, establishing server-side lookups directly to Clerk to confirm entitlements.
  - Created entitlement route guards (`isPro()`, `isCompanyPro()`) to protect premium API actions and hide premium tabs on the company dashboard.
* **Learning Takeaway:** Ensure billing lookups run server-side and fail closed. Web applications that check subscriptions on the frontend are vulnerable to client-side manipulation.

---

### Milestone 4: Architectural Self-Healing & Robustness
* **Commit:** `ecd4ab4` (Self-heal missing users rows...)
* **Key Files:** `convex/users.ts`
* **What Happened:**
  - Resolved a common bug in Clerk + Convex systems: when a user joins, the Clerk webhook to sync the user table might lag, resulting in errors during instant user actions.
  - Refactored mutation queries to gracefully **self-heal**. If Convex processes an authenticated mutation but can't find a corresponding user ID row, it inserts a new row dynamically on-the-fly instead of throwing a generic "Not authenticated" error.
* **Learning Takeaway:** Design robust systems. Designing database mutations to dynamically repair missing rows yields a seamless UX with fewer edge-case crashes.

---

### Milestone 5: Seamless Navigation, Global Search, and Typeahead
* **Commits:** `cef4d32` (Company onboarding...), `0d04d72` (Global search...), and `372edde` (Jobs search results...)
* **Key Files:** `convex/search.ts`, `components/layout/top-nav.tsx`, `components/jobs/search-input.tsx`
* **What Happened:**
  - Integrated global search to execute lightning-fast queries across jobs, companies, and people in a single typeahead dropdown.
  - Implemented standard Clerk onboarding steps. When a company signs up, it is native to Clerk's create/join organization flow before requiring company-profile details.
* **Learning Takeaway:** Simplify user onboarding by using standard auth-platform flows rather than coding multi-step signup portals from scratch.

---

### Milestone 6: Dedicated Routing and Lazy Loading over Modal-Hell
* **Commits:** `cb9ab51` (Replace applicant modal...) and `8bc72c6` (Job posts: expandable accordion...)
* **Key Files:** `app/(app)/company/applicants/[id]/page.tsx`, `components/company/job-post-list.tsx`
* **What Happened:**
  - Replaced bulky and slow modal popups (modals that fetch candidate detail summaries inside state-heavy windows) with clean, dedicated routes (`/company/applicants/[id]`).
  - Implemented lazy-loaded accordions for job listings on the hiring dashboard. When clicked, the accordion expands and lazily queries candidates assigned to that specific job ID, maximizing performance.
* **Learning Takeaway:** Prefer separate, explicit URLs for key details rather than putting all candidate interactions inside overlay modals. It yields better performance, cleaner code, and allows deep-linking for direct sharing among hiring teams.

---

### Milestone 7: Advanced Subagent Orchestration & Instruction Scoping
* **Commit:** `20aed99` (Enhance agent instructions...)
* **Key Files:** `agent/instructions.md`, `agent/subagents/job-scout/agent.ts`, `components/ai/agent-chat.tsx`
* **What Happened:**
  - Updated the agent behavior model to use a **manager-to-specialist** architecture.
  - Programmed the global agent to ask exactly one high-leverage question to gather requirements before kicking off tasks.
  - Delegated heavy processing (e.g., job analysis, profile polishing) to specialized subagents.
  - Configured the UI to display live feedback showing which subagent is currently handling a task, making the agentic workflow transparent and intuitive.
* **Learning Takeaway:** Breaking down a complex LLM process into smaller sub-tasks handled by specialized subagents produces higher-quality outputs with fewer hallucinations.

---

## 💡 How to Replicate and Learn From This Method

When building your next full-stack project, follow this progression to ensure success:

1. **Establish the Skeleton:** Initialize Next.js, format your packages, and configure Tailwind.
2. **Build the Database Layer:** Write a firm DB schema (e.g. Convex) and define your indexes.
3. **Protect Routes First:** Build in authentication middleware to protect non-public pages before writing complex views.
4. **Iterate with Static Features:** Build your feed, posting interfaces, and standard profiles first.
5. **Layer in Billing Constraints:** Configure payment integrations (e.g., Clerk Billing) and protect routes using server-side entitlement checks.
6. **Inject AI Agents Safely:** Create your agent system, isolate its tools, and ensure Human-in-the-Loop approvals protect database writes.
7. **Optimize for UX:** Switch from modals to page routes, lazy-load heavy lists, and use dedicated subagents to solve complex tasks.

Following this development sequence will help you build reliable, highly scalable, and production-ready applications!