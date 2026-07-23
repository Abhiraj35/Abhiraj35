# Product Blueprint & Architectural Design: JobSync AI
## Federated Job-Scout & Match Orchestrator

This document provides a highly detailed, comprehensive, and production-grade product blueprint for **JobSync AI** — a platform that lets users link their target job portals, aggregates matches via secure, distributed scraping agents, and compiles structured match reports (CSV, PDF, Interactive Tables).

We discuss the **Product Vision, Monetization Strategy, Architectural Flow, Anti-Bot Bypassing techniques, and full Database Schemas** so you can confidently build, pitch, and scale this into a successful product.

---

## 🚀 1. The Product Vision & Market Positioning

### The Problem
Finding a job is fragmented. Candidates waste hours daily checking LinkedIn, Indeed, Glassdoor, ZipRecruiter, and specialized niche boards.
*   **Existing Aggregators (like Indeed):** Flood users with outdated, generic, and uncurated listings.
*   **The Match Problem:** Candidates apply to hundreds of roles without knowing if their resume matches the applicant tracking systems (ATS) guidelines, resulting in high rejection rates.

### The Solution: JobSync AI
An AI-driven dashboard that acts as a **federated agentic search engine**.
1.  **Federated Portals:** The user links or activates target portals (e.g., LinkedIn, Indeed, local boards).
2.  **Autonomous Scraping Agents:** The system schedules lightweight headless scrapers.
3.  **ATS-Matching Engine:** It parses the user's resume, computes a semantic `% match score` for every fetched job description, and identifies key missing skills.
4.  **Exportable Reports:** Matches are formatted as interactive tables with export options (CSV, PDF) and direct application links.

### Monetization Strategy
*   **Free Tier:** Links up to 2 portals, scrapes once every 24 hours, and returns up to 10 matching jobs per run.
*   **Pro Tier ($12/month - B2C):** Links unlimited portals, runs real-time on-demand scrapes, unlocks the Vercel Eve AI Agent for resume tailoring, and exports unlimited PDFs/CSVs.
*   **Enterprise Recruiter Tier (B2B):** Organizations purchase licenses to reverse-search the candidate database based on active job openings.

---

## 🏗️ 2. High-Level Systems Architecture

We implement a highly decoupled, modern, and cost-effective system structure:

```
┌────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND                                  │
│                 Next.js 16 App Router (React 19)                       │
│    (Monaco Editor, Interactive Match Dashboard, Export Action Hooks)   │
└──────────────────┬───────────────────┬───────────────────┬─────────────┘
                   │                   │                   │
             (HTTPS / Auth)      (Reactive Sync)     (Trigger Run)
                   │                   │                   │
                   ▼                   ▼                   ▼
┌──────────────────────┐  ┌──────────────────────┐  ┌────────────────────┐
│      AUTH & BILLING  │  │   REACTIVE BACKEND   │  │   AI AGENT CHIPS   │
│        Clerk SaaS    │  │     Convex Cloud     │  │  Vercel Eve Agent  │
│                      │  │                      │  │                    │
│ • User Accounts      │  │ • Real-time DB Sync  │  │ • Job Scout Agent  │
│ • Pro Plans          │◀─┼─• File Storage       │◀─┼─• Skill Matcher    │
│                      │  │ • Vector Search      │  │ • Automated PDF/CSV│
└──────────────────────┘  └──────────────────────┘  └─────────┬──────────┘
                                                              │
                                                        (Launches VM)
                                                              │
                                                              ▼
                                                    ┌────────────────────┐
                                                    │  Microsandbox      │
                                                    │  Local MicroVM     │
                                                    │                    │
                                                    │ • Puppeteer Scraper│
                                                    │ • Cloudflare Bypass│
                                                    │ • Raw HTML Parser  │
                                                    └────────────────────┘
```

1.  **Next.js 16 (UI Shell):** Displays an interactive dashboard with a match comparison slider, filterable datatable (using TanStack Table), and export actions.
2.  **Clerk (Multi-Tenant Auth & B2C Billing):** Handles onboarding, profile management, and Gates the Pro plan subscription tier.
3.  **Convex (Reactive Storage & State Engine):**
    *   Holds user profiles, linked portal records, raw job listings, and vector embeddings.
    *   Handles real-time sync so the user sees results populating their screen live as scrapers finish.
4.  **Microsandbox (Distributed Execution):** Runs Puppeteer/Playwright scripts inside lightweight, isolated MicroVMs to execute web scrapers safely without overloading your host system.

---

## 🕵️ 3. The Scraping Pipeline: Bypassing Anti-Bot Walls

Web scraping large portals (LinkedIn, Indeed) is challenging due to security firewalls (Cloudflare, PerimeterX). To build a reliable product, implement this hybrid approach:

```
┌──────────────────────────────────────────────────────────────────────┐
│                      HYBRID SCRAPING STRATEGY                        │
└──────────────────┬────────────────────────────────┬──────────────────┘
                   │                                │
        [Method A: Public APIs]          [Method B: Headless MicroVM]
                   │                                │
         • Adzuna API / Jooble API        • Puppeteer Extra Stealth
         • High speed, 100% legal         • Bypasses Cloudflare
         • Clean structured JSON data     • Simulates human cursor movements
```

### Techniques for Headless Scrapers
*   **Puppeteer-Extra-Stealth:** Evades detection by masking headless attributes (navigator.webdriver, chrome plugins list, WebGL fingerprints).
*   **Rotating Residential Proxies:** Routes requests through residential IPs (using services like Bright Data or Oxylabs) to prevent rate-limiting.
*   **User-Agent Spofing:** Simulates real browser agents and randomizes screen resolutions.

### Legal & Ethical Guardrails
*   **Respect `robots.txt`:** Include delay parameters to avoid overloading target servers.
*   **Fallback to Public APIs:** Use official APIs (such as Adzuna, Jooble, or USAJobs) as the primary data source, reserving scrapers for custom niche portals.
*   **Local Scrapes:** Let Pro users run scrapes locally inside their own Client-side sandbox. This shifts the scraping footprint to their local IP, reducing your server operating costs to zero!

---

## 🗄️ 4. Concrete Database Schema

Implement this schema inside `convex/schema.ts` to manage users, linked portals, jobs, matches, and report logs:

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  users: defineTable({
    clerkId: v.string(),
    name: v.string(),
    email: v.string(),
    skills: v.array(v.string()),         // Extracted skills from resume
    resumeText: v.optional(v.string()),   // Cleaned raw resume text
  }).index("by_clerkId", ["clerkId"]),

  linkedPortals: defineTable({
    userId: v.id("users"),
    portalName: v.string(),              // "LinkedIn", "Indeed", "Custom"
    url: v.string(),                     // Link to search portal
    status: v.string(),                  // "connected", "scraping", "failed"
    lastScrapedAt: v.optional(v.number()),
  }).index("by_user", ["userId"]),

  rawJobs: defineTable({
    portalId: v.id("linkedPortals"),
    title: v.string(),
    company: v.string(),
    description: v.string(),
    location: v.string(),
    applyUrl: v.string(),
    sourceWebsite: v.string(),
    postedAt: v.number(),
    embedding: v.optional(v.array(v.float64())), // Vector embedding of job description
  }).vectorIndex("by_embedding", {
    vectorField: "embedding",
    dimensions: 1536,
  }),

  jobMatches: defineTable({
    userId: v.id("users"),
    jobId: v.id("rawJobs"),
    matchScore: v.number(),              // Percentage score (0 - 100)
    matchedSkills: v.array(v.string()),
    missingSkills: v.array(v.string()),
    status: v.string(),                  // "unread", "saved", "applied"
  })
    .index("by_user_score", ["userId", "matchScore"])
    .index("by_user_job", ["userId", "jobId"]),

  reportExports: defineTable({
    userId: v.id("users"),
    format: v.string(),                  // "CSV", "PDF"
    fileStorageId: v.id("_storage"),      // Convex internal file storage ID
    generatedAt: v.number(),
  }).index("by_user", ["userId"]),
});
```

---

## 🤖 5. The AI Match & Data Processing Flow

Once a scraper finishes, the raw job listing must go through the **Match and Vector Pipeline**:

```
┌──────────────┐      Convert Description      ┌──────────────┐      Compute Cosine      ┌──────────────┐
│ Raw Scraping │──────────────────────────────▶│ OpenAI Embed │─────────────────────────▶│Convex Vector │
│ HTML         │                               │ AI Gateway   │                          │Search Engine │
└──────────────┘                               └──────────────┘                          └──────┬───────┘
                                                                                                │
                                                                                      Matches with Resume text
                                                                                                │
                                                                                                ▼
                                                                                         ┌──────────────┐
                                                                                         │  JobMatches  │
                                                                                         │  Database    │
                                                                                         └──────────────┘
```

### A. Convert Job Description into Semantic Vector Embeddings
To avoid relying on exact keyword matches, generate a semantic vector embedding of the job description using OpenAI's `text-embedding-3-small` model:
```typescript
// convex/jobs.ts
import { action } from "./_generated/server";
import { v } from "convex/values";

export const generateJobEmbedding = action({
  args: { jobId: v.id("rawJobs"), text: v.string() },
  handler: async (ctx, args) => {
    // 1. Fetch embedding vector
    const response = await fetch("https://api.openai.com/v1/embeddings", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${process.env.OPENAI_API_KEY}`,
      },
      body: JSON.stringify({
        input: args.text,
        model: "text-embedding-3-small",
      }),
    });

    const res = await response.json();
    const embedding = res.data[0].embedding;

    // 2. Save vector to the raw jobs table in Convex
    await ctx.runMutation(api.jobs.updateJobEmbedding, {
      jobId: args.jobId,
      embedding,
    });
  },
});
```

### B. Compute Score & Evaluate Matches
Calculate the similarity score between the user's resume vector and the job description vector to get a semantic percentage score. Then use Claude (or GPT) to quickly extract missing skills:
```typescript
// convex/jobs.ts
import { mutation } from "./_generated/server";
import { v } from "convex/values";

export const calculateMatchDetails = mutation({
  args: { userId: v.id("users"), jobId: v.id("rawJobs") },
  handler: async (ctx, args) => {
    const user = await ctx.db.get(args.userId);
    const job = await ctx.db.get(args.jobId);
    if (!user || !job) return;

    // Extract overlapping terms to calculate matched vs missing skills
    const userSkills = new Set(user.skills.map(s => s.toLowerCase()));
    const jobSkills = job.skillsRequired.map(s => s.toLowerCase());

    const matchedSkills = jobSkills.filter(s => userSkills.has(s));
    const missingSkills = jobSkills.filter(s => !userSkills.has(s));

    // Calculate structural match percentage
    const matchScore = Math.round((matchedSkills.length / Math.max(jobSkills.length, 1)) * 100);

    await ctx.db.insert("jobMatches", {
      userId: args.userId,
      jobId: args.jobId,
      matchScore,
      matchedSkills,
      missingSkills,
      status: "unread",
    });
  },
});
```

---

## 📄 6. Compiling Structured Report Exports (CSV & PDF)

Once matches are saved, the user can download their report. To keep costs low, compile files **on-the-fly inside Next.js Server Actions** or Edge Functions, saving results directly to **Convex File Storage**:

### A. CSV Generation (Low CPU Overhead)
Convert matching job arrays to standard string arrays and serve them using a basic Next.js endpoint:
```typescript
// app/api/export/csv/route.ts
import { NextResponse } from "next/server";

export async function POST(req: Request) {
  const { matches } = await req.json();

  let csvContent = "Job Title,Company,Match Score %,Location,Source,Apply Link\n";
  matches.forEach((m: any) => {
    csvContent += `"${m.title}","${m.company}",${m.score},"${m.location}","${m.source}","${m.applyUrl}"\n`;
  });

  return new NextResponse(csvContent, {
    headers: {
      "Content-Type": "text/csv",
      "Content-Disposition": "attachment; filename=job_match_report.csv",
    },
  });
}
```

### B. PDF Generation (High Visual Polish)
Use `@react-pdf/renderer` to compile beautiful, printable job match summaries inside Next.js Server Components, then stream the PDF binary directly back to the user.

---

## 🛠️ 7. Your Step-by-Step Implementation Checklist

Follow this build checklist to construct **JobSync AI** from scratch:

1.  **Configure App Shell:** Scaffold Next.js 16 and authenticate it using **Clerk CLI**.
2.  **Establish Schema:** Build `convex/schema.ts` containing the five tables outlined above.
3.  **Setup Scrapers:** Write lightweight scraping scripts inside `agent/tools/` using Puppeteer-Extra-Stealth.
4.  **Wire up Embeddings:** Set up your Vercel AI Gateway key and write the embeddings calculation actions.
5.  **Build the Front-End Datatable:** Use `lucide-react` icons and TanStack Tables to build an interactive, filterable match grid. Add PDF/CSV download action buttons.
6.  **Add User Customization:** Create a resume upload interface, using pdf-parse on the server to extract text and populate the user's skill set automatically.

This project demonstrates strong capabilities in system architecture, microVM isolation, web standards, and secure data pipelines — making it a massive advantage in any software engineering interview!