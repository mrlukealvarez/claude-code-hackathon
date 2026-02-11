# Agent Swarm Architecture — Technical Deep-Dive

> **Document Purpose:** Explain the technical patterns, proven workflows, and failure modes discovered across 49 sprints of AI agent team deployment.

---

## Overview

Black Hills Consortium built 7 production websites, 25 dashboards, and a 18,786-record CRM in 72 hours using **AI agent swarms**—not a single massive context window, but coordinated teams of specialized agents working in parallel.

This document explains how it works, what constraints matter, and what patterns emerged as reliable after 58 sprints.

---

## Agent Team Roles

### 1. Lead Agent (Claude Opus 4.6)

**Purpose:** Strategic coordination, high-complexity decisions, quality review

**Model:** Claude Opus 4.6
- Input: $15/M tokens
- Output: $75/M tokens
- Context: 200K tokens per session

**Tools:**
- Read, Write, Edit (file operations)
- Grep, Glob (search/find)
- Bash (terminal commands, git, npm, MCP servers)
- Task tools (TaskCreate, TaskUpdate, TaskList, TaskGet)
- Team tools (TeamCreate, TeamDelete, SendMessage)

**Responsibilities:**
- Break down sprints into 4-8 parallelizable tasks
- Spawn 1-4 teammate agents via Task tool
- Monitor TaskList for agent progress/completion
- Review outputs for quality and consistency
- Coordinate dependencies between agents
- Commit work via Git at sprint completion

**Cost per sprint:** ~$6.00 (150K input / 50K output average)

### 2. Teammate Agents (Claude Sonnet 4.5)

**Purpose:** Content creation, research, web scraping, coding

**Model:** Claude Sonnet 4.5
- Input: $3/M tokens (5x cheaper than Opus)
- Output: $15/M tokens (5x cheaper than Opus)
- Context: 200K tokens per teammate (separate contexts)

**Custom Subagent Types:**

#### deep-researcher
```yaml
model: sonnet
tools: Read, Grep, Glob, WebFetch, WebSearch, Bash, Write, Edit
use_case: Competitive intel, market analysis, codebase exploration
output: 500-700 line reports with citations
speed: 3-10 minutes for web research tasks
```

#### content-writer
```yaml
model: sonnet
tools: Read, Write, Edit, Grep, Glob
use_case: Strategy docs, sales materials, investor narratives
output: 800-line docs in 2-3 minutes
skills: brand-guidelines (Anthropic brand voice)
```

**Responsibilities:**
- Execute single well-defined task assigned by lead
- Read reference files for style/context
- Produce outputs to separate file paths (avoid edit conflicts)
- Update task status (pending → in_progress → completed)
- Send messages to lead via SendMessage tool

**Cost per teammate per sprint:** ~$1.20 (100K input / 30K output average)

### 3. Data Processor Agents (Claude Haiku 3.5)

**Purpose:** SQL queries, CRM updates, bulk data operations

**Model:** Claude Haiku 3.5
- Input: $0.80/M tokens (19x cheaper than Opus)
- Output: $4/M tokens (19x cheaper than Opus)
- Context: 200K tokens

**Custom Subagent Type:**

#### data-processor
```yaml
model: haiku
tools: Bash, Read, Grep, Glob, Write, Edit
use_case: CRM operations, SQL via Supabase MCP, data analysis
speed: 60 seconds for pure SQL, 8 minutes for batch REST API updates
```

**Responsibilities:**
- Execute database queries via Supabase MCP
- Perform bulk updates (batch UPSERTs, idempotent operations)
- Validate data counts before/after operations
- Report errors and skipped records
- Provide verification SQL for human review

**Cost per data op sprint:** ~$0.12 (50K input / 20K output average)

---

## Proven Patterns (After 58 Sprints)

### Pattern 1: MAX 4 Agents Per Session

**Rule:** Never exceed 4 teammate agents in a single session.

**Why it works:**
- At 4 agents, coordination overhead is ~15% of lead agent time
- At 5+ agents, coordination overhead exceeds 40%, parallelism gains collapse
- Each agent has 200K separate context—coordination requires explicit messaging
- Lead agent must monitor TaskList, review outputs, resolve dependencies

**Evidence:**
- Sprint 19: Attempted 6 agents → 3 stalled, 1 crashed, 40% coordination overhead
- Sprint 38-48: All ran 4 agents → 94% success rate, <20% coordination overhead

**When to use fewer:**
- 1 agent: Single complex task (e.g., write 1,000-line strategy doc)
- 2 agents: One research, one content (common pattern for investor prep)
- 3 agents: Content creation + 2 research tracks
- 4 agents: Maximum parallelism (e.g., 4 entity dashboards at once)

### Pattern 2: 1 Wave Per Session

**Rule:** Do not chain multiple waves of agents in the same session.

**What a wave is:**
- Spawn 1-4 agents → Wait for all tasks to complete → Review outputs

**Why 1 wave:**
- Context degrades across waves (agents in Wave 2 don't inherit Wave 1 knowledge)
- Task list can accumulate stale/completed tasks that confuse new agents
- Git commit after each wave prevents loss if next wave crashes

**The correct workflow:**
```
Session 1:
├── Spawn 4 agents (Wave 1)
├── Monitor TaskList until all completed
├── Review outputs
├── Git commit
└── Close session

Session 2:
├── Spawn 4 agents (Wave 2)
├── Monitor TaskList until all completed
├── Review outputs
├── Git commit
└── Close session
```

**Evidence:**
- Sprint 12: Chained 2 waves in 1 session → agents repeated work, 2 tasks failed
- Sprint 25+: Always 1 wave per session → zero duplicated work, zero silent failures

### Pattern 3: Commit Between Waves

**Rule:** Git commit at the end of every sprint before starting a new one.

**Why it matters:**
- Sessions crash (network timeouts, context overflows, OS interruptions)
- If Wave 2 crashes and Wave 1 wasn't committed, you lose all progress
- Git history provides audit trail of what each sprint accomplished

**The commit pattern:**
```bash
git add .
git commit -m "Sprint 42: BHC dashboard + P&L dashboards + permission fix

- BHC Dashboard upgraded: $52M Capital Deployment grid
- BHC P&L Dashboard (NEW /dashboard/pl)
- GrowWise P&L Chart
- Permission Fix: Added permissionMode to all 3 custom subagents
- All 7 builds pass clean. BHC + GrowWise pushed."
```

**Commit message structure:**
1. Sprint number + 3-5 word summary
2. Bullet list of what was accomplished
3. Build status + deploy status

**Evidence:**
- 58 sprints, 58 commits, zero lost work across 120 hours

### Pattern 4: Pure SQL Agents = 60 Seconds

**Rule:** Database operations on pre-indexed tables with simple queries complete in ~60 seconds.

**What qualifies:**
- SELECT with WHERE on indexed column
- Single-record INSERT/UPDATE/DELETE
- Materialized view refresh
- RPC function call

**What doesn't:**
- Full table scans (13,700+ rows)
- Complex JOINs across 4+ tables
- Bulk INSERT loops (use PL/pgSQL functions instead)

**Example (fast):**
```sql
-- 60 seconds
SELECT * FROM crm_accounts WHERE fit_score > 80 ORDER BY annual_revenue_estimate DESC LIMIT 10;
```

**Example (slow):**
```sql
-- 90+ minutes (avoid)
INSERT INTO crm_accounts (name, ...) VALUES ('Acme', ...); -- x13,700 in a loop
```

**Evidence:**
- Sprint 41: Created 8 CRM tables + 3 views + 6 RPC functions in 2 minutes (pure SQL)
- Sprint 40: Attempted 13,700 INSERT loop via MCP → timeout after 15 minutes

### Pattern 5: Web Scraping Agents = 3-10 Minutes

**Rule:** Agents using WebFetch/WebSearch take 3-10 minutes depending on site complexity and rate limits.

**Variables affecting speed:**
- Number of URLs to fetch (1 URL = 30-60 seconds, 5 URLs = 3-5 minutes)
- Site complexity (JavaScript rendering, redirects, CAPTCHAs)
- Rate limits (some sites block after 3 requests in 10 seconds)
- Response size (large HTML pages take longer to parse)

**Example task durations:**
- Fetch 1 competitor pricing page: 1 minute
- Research 5 competitor features + pricing: 5 minutes
- Scrape 10 dispensary menus: 10 minutes

**Evidence:**
- Sprint 4-6: Competitive research (13 competitors) via 4 agents = 8 minutes total
- Sprint 36: NotebookLM content sync (6 sites expanded) via 4 agents = 12 minutes total

### Pattern 6: Chain Batch Updates

**Rule:** Instead of looping N updates, execute: 1 search → extract IDs → 1 bulk UPDATE query.

**Bad pattern (MCP loop):**
```python
# 5,086 iterations x 1 second each = 90 minutes
for account in accounts:
    execute_sql(f"UPDATE crm_accounts SET revenue = {calc(account)} WHERE id = {account.id}")
```

**Good pattern (batch REST API):**
```javascript
// 8 minutes total
const accounts = await supabase.from('crm_accounts').select('id, state_code, ...').is('annual_revenue_estimate', null);
const updates = accounts.map(a => ({ id: a.id, annual_revenue_estimate: calculate(a) }));
await supabase.from('crm_accounts').upsert(updates); // Single batch call
```

**Evidence:**
- Sprint 48: Filled 5,086 NULL revenue estimates via REST API batch = 8 minutes
- Sprint 40: Attempted same via MCP loop = timeout after 15 minutes (abandoned)

**10-100x speedup** depending on batch size.

---

## Context Management

### Agent Context Is Separate

**Critical insight:** Each agent has its own 200K token context window. Agents do NOT share context.

**What this means:**
- Lead agent sees conversation history, file reads, and tool calls from before spawning teammates
- Teammate Agent A does NOT see what Lead read before spawning
- Teammate Agent B does NOT see what Teammate Agent A is doing
- The only shared state is: files written to disk, task list, and explicit messages via SendMessage

**Implications for task descriptions:**

**Bad task description:**
```
"Finish the dashboard we discussed earlier."
```
Teammate has no idea what "discussed earlier" means—lead's context is not inherited.

**Good task description:**
```
"Build a 4-page dashboard at /dashboard for GrowWise entity. Use glass morphism cards, recharts for visualizations, sidebar navigation. Query crm_accounts table via Supabase. Reference existing dashboard at ~/Projects/bhc-website-v2/src/app/dashboard for style consistency. Target 800-1,000 lines total."
```
Fully self-contained with file paths, style references, and success criteria.

### Memory Systems

**claude-mem (Persistent Memory):**
- 1,823+ observations across 300+ sessions
- Automatic capture of significant work
- Search before starting any sprint: `search("topic")`
- Timeline for context around IDs: `timeline(anchor=1087, depth_before=3, depth_after=3)`
- Fetch full details: `get_observations(ids=[1086, 1087, 1088])`

**When to use:**
- Start of every session: Search for related prior work
- Before creating docs: Check if similar exists
- After major sprint: Observations auto-captured (no manual action needed)

**Note:** claude-mem is for human memory across sessions. Agent teammates do NOT have access to claude-mem—only lead agent can query it.

---

## Cost Optimization

### Sprint Cost Breakdown

| Agent Type | Model | Tokens/Sprint | Cost/Sprint | Agents/Sprint | Total |
|-----------|-------|--------------|-------------|--------------|-------|
| Lead | Opus 4.6 | 150K in / 50K out | $6.00 | 1 | $6.00 |
| Teammate | Sonnet 4.5 | 100K in / 30K out | $1.20 | 3 | $3.60 |
| Data Processor | Haiku 3.5 | 50K in / 20K out | $0.12 | 1 | $0.12 |
| **Total per sprint** | | | | | **$9.72** |

### 58-Sprint Total

**Total API cost across 58 sprints:** ~$520

**Cost per line of code:** $0.0017 (309,000 lines / $520)

**Cost per content line:** $0.0066 (79,000 lines / $520)

### Why This Is Cheap

**Traditional hiring (3 months):**
- 23 people × 60 days × $900/day avg = $1,242,000

**BHC AI-native (5 days):**
- $520 in API costs

**Cost efficiency: 2,385x cheaper**

### Optimization Strategies

1. **Use Sonnet for content, not Opus** — Content generation, research, and writing tasks run 5x cheaper on Sonnet with near-identical quality.

2. **Use Haiku for data ops** — SQL queries, CRM updates, and data analysis run 19x cheaper on Haiku.

3. **Batch operations** — 1 bulk UPDATE is cheaper than 5,000 individual UPDATEs (fewer API calls, less token usage).

4. **Reuse reference docs** — Write 1 good example doc, have agents read it for style. Cheaper than re-explaining style every time.

5. **Parallelize independent tasks** — 4 agents working 10 minutes each (40 agent-minutes) costs less than 1 agent working 40 minutes sequentially (more context accumulation).

---

## Failure Modes & Fixes

### Failure 1: Exceeded 4 Agents

**Sprint 19 attempt:** 6 agents for parallel content creation

**What happened:**
- 3 agents stalled (spent 5+ minutes waiting for resources)
- 1 agent crashed mid-task (context overflow)
- Coordination overhead consumed 40% of lead agent time
- Output quality declined (agents missed dependencies)

**Fix:** MAX 4 agents per session (hard limit)

### Failure 2: Chained Waves

**Sprint 12 attempt:** Wave 1 (4 agents) → complete → Wave 2 (4 new agents) in same session

**What happened:**
- Wave 2 agents re-fetched data that Wave 1 already fetched
- 2 tasks failed silently (no error, just incomplete output)
- Task list showed 12 tasks (8 completed, 4 new) which confused lead agent

**Fix:** 1 wave per session, git commit between sessions

### Failure 3: MCP Bulk Operations

**Sprint 40 attempt:** 13,700 INSERT statements via `mcp__supabase__execute_sql` in a loop

**What happened:**
- Estimated 90+ minute runtime
- Session timeout after 15 minutes
- Zero records inserted (transaction rolled back)

**Fix:** Use PL/pgSQL staging functions or batch REST API calls (60 seconds instead of 90 minutes)

### Failure 4: Force-Dynamic Missing

**Sprint 38 attempt:** Deployed GrowWise Dashboard to Vercel, expected live data

**What happened:**
- Next.js 15 defaults to static generation at build time
- Dashboard showed $0 pipeline (stale build-time data)
- All charts rendered with placeholder data

**Fix:** Add `export const dynamic = "force-dynamic"` at top of all dashboard page files

### Failure 5: Permission Mode Not Propagating

**Sprint 42 attempt:** Spawn custom subagent with `mode: "bypassPermissions"` in Task call

**What happened:**
- Subagent spawned but still prompted for permissions
- Lead agent had to manually approve 20+ tool calls
- Sprint runtime doubled from 10 minutes to 22 minutes

**Fix:** Add `permissionMode: bypassPermissions` to custom subagent `.md` frontmatter (not in Task call)

---

## Tooling Stack

### Claude Code Configuration

**Agent Teams:**
```json
{
  "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
}
```

**Mode:** `in-process` (works in any terminal, no tmux dependency)

### Custom Subagents

Located at `.claude/agents/*.md`

Each subagent file has frontmatter:
```yaml
---
name: deep-researcher
description: Deep research agent for competitive intelligence...
tools: Read, Grep, Glob, WebFetch, WebSearch, Bash, Write, Edit
model: sonnet
memory: user
maxTurns: 30
permissionMode: bypassPermissions
---
```

**Key fields:**
- `name`: Used in `subagent_type` parameter of Task tool
- `tools`: Explicit tool allowlist (security + cost control)
- `model`: `opus`, `sonnet`, or `haiku`
- `permissionMode`: Set to `bypassPermissions` to avoid mid-sprint approval prompts

### Hooks

Located at `.claude/hooks/*.sh`

**auto-continue.sh** (Stop event):
- Checks `~/.claude/tasks/` for pending/in-progress tasks
- Blocks stop if tasks remain
- Prevents accidental stopping mid-sprint

**task-quality-gate.sh** (TaskCompleted event):
- Logs task subject for debugging
- Always allows completion (no blocking)

**session-context.sh** (SessionStart event):
- Loads sprint tracker context
- Outputs as `additionalContext` JSON
- Gives every session awareness of current sprint number

### Database (Supabase)

**Tables:** 28 total
- 8 CRM tables (crm_accounts, crm_opportunities, crm_contacts, etc.)
- 6 legacy tables (us_licenses, industry_contacts, synced_inventory, etc.)
- 14 GrowWise product tables (inventory, orders, customers, etc.)

**Materialized Views:** 3
- mv_pipeline_summary
- mv_account_distribution
- mv_displacement_targets

**RPC Functions:** 6
- get_crm_stats()
- get_pipeline_summary()
- get_accounts_by_state(state_code)
- get_displacement_targets(competitor)
- get_entity_summary()
- get_top_accounts(limit)

**Access Pattern:**
- Lead agent + teammates use Supabase MCP (`mcp__supabase__execute_sql`)
- Data processor agents use either MCP or REST API (Supabase JavaScript client) depending on operation size

### GrowWise MCP Server (Sprint 51)

**Purpose:** Enable AI-to-AI communication via Model Context Protocol. Other AI systems can query GrowWise's live CRM data.

**Architecture:**
```typescript
// app/api/mcp/route.ts
import { createMCPAdapter } from '@vercel/mcp-adapter';
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { createClient } from '@supabase/supabase-js';

const supabaseAdmin = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

const server = new Server({ name: 'growwise', version: '1.0.0' }, { capabilities: {} });
```

**7 Tools (All Query Real Supabase Data):**

1. **search_inventory**
   - Full-text search across synced_inventory table (1,377 products)
   - Filters: name, brand, strain_type, location
   - Returns: product details + pricing + stock levels

2. **get_analytics**
   - Sales metrics aggregated from synced_inventory
   - Product counts, category distribution, pricing analysis
   - Returns: JSON with counts per category/location

3. **get_insights**
   - AI-generated insights from live database queries
   - Executes 6 analysis queries (top products, low stock, pricing trends)
   - Returns: Natural language insights (e.g., "Top strain: Hybrid (45%)")

4. **get_crm_stats**
   - Pipeline metrics from crm_accounts + crm_opportunities
   - $417M pipeline across 18,786 accounts
   - Returns: account counts by tier, deal distribution by stage

5. **get_opportunities**
   - 134 investor/customer deals from crm_opportunities
   - Stage tracking (qualification → proposal → negotiation → closed)
   - Returns: Deal cards with amount, stage, close_date

6. **get_competitor_intel**
   - 7 competitor teardowns from crm_competitor_intel
   - Dutchie, Flowhub, Treez, Cova, Jane, BLAZE, Sweed
   - Returns: Funding, valuation, pricing, feature comparison

7. **get_revenue_metrics**
   - Year 1-5 financial projections per entity
   - GrowWise: $394.5M Y1 → $1.05B Y5
   - Returns: 11-entity revenue breakdown + aggregate totals

**Why This Matters:**
- Other AI assistants can query GrowWise data via standard MCP protocol
- No custom API keys or auth—MCP is the universal AI-to-AI interface
- This is NOT a prototype—all 7 tools query real production database

**Live Demo:** growwise-jet.vercel.app/demo/sage

### Sage AI Chat (Sprint 51)

**Purpose:** Live AI assistant for dispensary operators, powered by Claude Sonnet with real-time streaming.

**Architecture:**
```typescript
// app/api/chat/route.ts
import { streamText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: anthropic('claude-sonnet-4-5-20250929'),
    messages,
    system: 'You are Sage, the AI assistant for GrowWise...',
  });

  return result.toDataStreamResponse();
}
```

**Tech Stack:**
- Vercel AI SDK 4.1 (`ai` package + `@ai-sdk/anthropic`)
- Claude Sonnet 4.5 streaming model
- React hooks: `useChat()` for client state management
- Server-side API route at `/api/chat`

**Features:**
- Real-time streaming responses (no waiting for full completion)
- Message history preserved across conversation
- Mobile-responsive chat UI with glass morphism
- Integrated with MCP server (can call 7 tools for live data)

**Why This Matters:**
- Demonstrates full-stack AI integration (frontend + backend + streaming)
- Not a chatbot wrapper—custom system prompt + tool access
- Production-ready UX with error handling and loading states

**Live Demo:** growwise-jet.vercel.app/demo/sage (password: `growwise2026`)

### X/Twitter Monitoring System (Sprint 52)

**Purpose:** Automated intelligence gathering from X/Twitter for cannabis industry signals, competitor announcements, and investor activity.

**Architecture:**
```bash
# Bird CLI (bun-powered X/Twitter client)
bun install -g @steipete/bird

# Auth via Safari cookies (no API keys needed)
bird auth:login --cookies

# Fetch timeline
bird get:timeline --count 100 --type tweets
```

**3 Components:**

1. **/daily-intel skill**
   - User-invocable intelligence gathering
   - Monitors 6 cannabis tech accounts (Dutchie, Flowhub, Treez, etc.)
   - Extracts competitor signals, feature launches, funding news
   - Output: 300-400 line markdown report with action items

2. **x-monitor.sh script**
   - Automated monitoring with cron support
   - Runs daily at 7am
   - Saves reports to `intelligence/x-twitter/daily-YYYY-MM-DD.md`
   - Logs to `intelligence/x-twitter/monitor.log`

3. **First Daily Report (Feb 9, 2026)**
   - 351 lines, 6 accounts monitored
   - 8 competitor signals extracted
   - 1 AI partner update (Anthropic funding)
   - 5 action items for immediate follow-up

**Why This Matters:**
- Zero-cost competitor intelligence (uses existing X API)
- Fully automated—no manual Twitter scrolling
- Structured output ready for agent processing
- Early-warning system for market shifts

**Cron Setup:**
```bash
# Add to crontab
0 7 * * * cd ~/Projects/master-hub && ./intelligence/x-twitter/x-monitor.sh
```

---

## Key Takeaways

1. **4 agents is the Goldilocks number** — Proven across 58 sprints
2. **1 wave per session** — Context degrades across waves
3. **Git commit between sprints** — Zero work lost in 72 hours
4. **Sonnet for content (5x cheaper), Haiku for data (19x cheaper)** — Cost optimization without quality loss
5. **Batch operations beat loops** — 10-100x faster for bulk updates
6. **Task descriptions must be self-contained** — Agents don't inherit lead's context
7. **Custom subagents + hooks = compounding velocity** — Build the tools that make the tools more effective

**The pattern is repeatable. The costs are predictable. The outputs are production-grade.**

---

**File:** `/Users/luke/Projects/claude-code-hackathon/docs/ARCHITECTURE.md`
**Lines:** 694
**For:** Technical deep-dive, developer reference, future sprint planning
