# Brief: Standalone MCP Server — New Project Setup

**For:** Antigravity  
**Effort:** ~1 hour  
**Result:** A dedicated MCP server at `alabs-mcp.vercel.app/api/mcp`, completely isolated from careersuccess-legacy

---

## What you're building

A brand-new minimal Next.js 15 project containing only the Google Ads MCP server. No pages, no UI — just one API route. It lives in its own GitHub repo and its own Vercel project so that LP deployments can never affect it.

---

## Step 1 — Create GitHub Repo

In the `dhaval-alabs` GitHub account, create a new **public** repository:

- **Name:** `alabs-mcp-server`
- **Description:** AnalytixLabs Google Ads MCP Server
- **Init with:** README only (no template)

---

## Step 2 — Set up the project files

Clone the new repo and create this exact file structure:

```
alabs-mcp-server/
├── package.json
├── next.config.ts
├── tsconfig.json
└── app/
    └── api/
        └── [transport]/
            └── route.ts
```

Copy each file exactly as provided below.

---

## File 1: `package.json`

```json
{
  "name": "alabs-mcp-server",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "next": "^15.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "mcp-handler": "latest",
    "@modelcontextprotocol/sdk": "1.26.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "@types/react": "^19.0.0",
    "typescript": "^5.0.0"
  }
}
```

---

## File 2: `next.config.ts`

```typescript
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  // Intentionally minimal — no experimental flags
}

export default nextConfig
```

---

## File 3: `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

---

## File 4: `app/api/[transport]/route.ts`

This is the entire server. Copy it exactly — do not modify.

```typescript
import { createMcpHandler } from 'mcp-handler'
import { z } from 'zod'

// IMPORTANT: must be a route-level export for Vercel timeout
export const maxDuration = 60

// Config
const CUSTOMER_ID = process.env.GOOGLE_ADS_CUSTOMER_ID ?? '4064995850'
const MCC_ID      = process.env.GOOGLE_ADS_MCC_ID      ?? '8910137241'
const API_VER     = process.env.GOOGLE_ADS_API_VERSION  ?? 'v19'
const GADS_URL    = `https://googleads.googleapis.com/${API_VER}/customers/${CUSTOMER_ID}/googleAds:search`

// OAuth
async function getAccessToken(): Promise<string> {
  const res = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      client_id:     process.env.GOOGLE_ADS_CLIENT_ID!,
      client_secret: process.env.GOOGLE_ADS_CLIENT_SECRET!,
      refresh_token: process.env.GOOGLE_ADS_REFRESH_TOKEN!,
      grant_type:    'refresh_token',
    }),
  })

  if (!res.ok) {
    const body = await res.text()
    throw new Error(`OAuth failed (HTTP ${res.status}): ${body.slice(0, 300)}`)
  }

  const data = (await res.json()) as { access_token?: string; error?: string; error_description?: string }
  if (!data.access_token) {
    throw new Error(`OAuth error: ${data.error ?? 'unknown'} — ${data.error_description ?? JSON.stringify(data)}`)
  }

  return data.access_token
}

// GAQL Runner
type Row = Record<string, unknown>

async function gaql(query: string): Promise<Row[]> {
  const token = await getAccessToken()

  const res = await fetch(GADS_URL, {
    method: 'POST',
    headers: {
      'Authorization':      `Bearer ${token}`,
      'developer-token':    process.env.GOOGLE_ADS_DEVELOPER_TOKEN!,
      'login-customer-id':  MCC_ID,
      'Content-Type':       'application/json',
    },
    body: JSON.stringify({ query }),
  })

  if (!res.ok) {
    const body = await res.text()
    throw new Error(`Google Ads API error (HTTP ${res.status}): ${body.slice(0, 500)}`)
  }

  const data = (await res.json()) as { results?: Row[] }
  return data.results ?? []
}

// Helpers
function dateRange(days: number) {
  const end   = new Date()
  const start = new Date(Date.now() - days * 86_400_000)
  const fmt   = (d: Date) => d.toISOString().slice(0, 10)
  return { start: fmt(start), end: fmt(end), today: fmt(end) }
}

function pick(obj: unknown, path: string): unknown {
  return path.split('.').reduce((acc, key) => (acc as Row)?.[key], obj)
}

function toInr(micros: unknown): number {
  return Number(micros ?? 0) / 1_000_000
}

function fmt(n: number, dp = 0): string {
  return n.toFixed(dp)
}

function divider(char = '─', len = 44) {
  return char.repeat(len)
}

// MCP Handler
const handler = createMcpHandler(
  (server) => {

    // Tool 1: Campaign Stats
    server.tool(
      'get_campaign_stats',
      'Get performance stats for all enabled campaigns. Returns clicks, conversions, cost and CPA.',
      {
        days:            z.number().default(30).describe('Lookback window in days'),
        campaign_filter: z.string().optional().describe('Filter by campaign name substring'),
      },
      async ({ days = 30, campaign_filter }) => {
        const { start, end } = dateRange(days)
        let where = `campaign.status = 'ENABLED' AND segments.date BETWEEN '${start}' AND '${end}'`
        if (campaign_filter) where += ` AND campaign.name LIKE '%${campaign_filter}%'`

        const rows = await gaql(`
          SELECT
            campaign.id,
            campaign.name,
            metrics.clicks,
            metrics.impressions,
            metrics.conversions,
            metrics.all_conversions,
            metrics.cost_micros
          FROM campaign
          WHERE ${where}
          ORDER BY metrics.cost_micros DESC
        `)

        const map = new Map<string, { name: string; clicks: number; conv: number; cost: number; impr: number }>()
        for (const row of rows) {
          const id   = String(pick(row, 'campaign.id') ?? 'x')
          const name = String(pick(row, 'campaign.name') ?? '')
          const cur  = map.get(id) ?? { name, clicks: 0, conv: 0, cost: 0, impr: 0 }
          cur.clicks += Number(pick(row, 'metrics.clicks')      ?? 0)
          cur.conv   += Number(pick(row, 'metrics.conversions') ?? 0)
          cur.cost   += toInr(pick(row, 'metrics.costMicros'))
          cur.impr   += Number(pick(row, 'metrics.impressions') ?? 0)
          map.set(id, cur)
        }

        const results = [...map.values()].sort((a, b) => b.cost - a.cost)
        if (!results.length) return { content: [{ type: 'text' as const, text: 'No campaign data found for this period.' }] }

        const lines = results.map(c => {
          const cpa = c.conv > 0 ? `₹${fmt(c.cost / c.conv)}` : 'N/A'
          return `• ${c.name}\n  Clicks ${c.clicks} | Conversions ${fmt(c.conv, 1)} | Spend ₹${fmt(c.cost)} | CPA ${cpa}`
        })

        return { content: [{ type: 'text' as const, text: `Campaign Stats — last ${days} days\n${divider()}\n${lines.join('\n\n')}` }] }
      }
    )

    // Tool 2: Keyword Stats
    server.tool(
      'get_keyword_stats',
      'Get keyword performance. Useful for finding high CPA or underperforming keywords.',
      {
        days:            z.number().default(30).describe('Lookback window in days'),
        campaign_filter: z.string().optional().describe('Filter by campaign name substring'),
        limit:           z.number().default(50).describe('Max number of keywords to return'),
        max_cpa_inr:     z.number().optional().describe('Only return keywords with CPA above this value in INR'),
      },
      async ({ days = 30, campaign_filter, limit = 50, max_cpa_inr }) => {
        const { start, end } = dateRange(days)
        let where = `campaign.status = 'ENABLED' AND ad_group_criterion.status = 'ENABLED' AND segments.date BETWEEN '${start}' AND '${end}'`
        if (campaign_filter) where += ` AND campaign.name LIKE '%${campaign_filter}%'`

        const rows = await gaql(`
          SELECT
            campaign.name,
            ad_group.name,
            ad_group_criterion.keyword.text,
            ad_group_criterion.keyword.match_type,
            metrics.clicks,
            metrics.impressions,
            metrics.conversions,
            metrics.cost_micros
          FROM keyword_view
          WHERE ${where}
          ORDER BY metrics.cost_micros DESC
          LIMIT ${limit * 10}
        `)

        const map = new Map<string, { kw: string; match: string; campaign: string; clicks: number; impr: number; conv: number; cost: number }>()
        for (const row of rows) {
          const kw       = String(pick(row, 'adGroupCriterion.keyword.text')      ?? '')
          const match    = String(pick(row, 'adGroupCriterion.keyword.matchType') ?? '')
          const campaign = String(pick(row, 'campaign.name')                       ?? '')
          const key      = `${campaign}|${kw}|${match}`
          const cur      = map.get(key) ?? { kw, match, campaign, clicks: 0, impr: 0, conv: 0, cost: 0 }
          cur.clicks += Number(pick(row, 'metrics.clicks')      ?? 0)
          cur.impr   += Number(pick(row, 'metrics.impressions') ?? 0)
          cur.conv   += Number(pick(row, 'metrics.conversions') ?? 0)
          cur.cost   += toInr(pick(row, 'metrics.costMicros'))
          map.set(key, cur)
        }

        let results = [...map.values()].sort((a, b) => b.cost - a.cost)
        if (max_cpa_inr !== undefined) {
          results = results.filter(k => {
            const cpa = k.conv > 0 ? k.cost / k.conv : Infinity
            return cpa > max_cpa_inr
          })
        }
        results = results.slice(0, limit)
        if (!results.length) return { content: [{ type: 'text' as const, text: 'No keyword data found for this period.' }] }

        const lines = results.map(k => {
          const cpa = k.conv > 0 ? `₹${fmt(k.cost / k.conv)}` : 'N/A'
          return `• [${k.match}] "${k.kw}"\n  Campaign: ${k.campaign}\n  Clicks ${k.clicks} | Conv ${fmt(k.conv, 1)} | Spend ₹${fmt(k.cost)} | CPA ${cpa}`
        })

        return { content: [{ type: 'text' as const, text: `Keyword Stats — last ${days} days (${results.length} keywords)\n${divider()}\n${lines.join('\n\n')}` }] }
      }
    )

    // Tool 3: Search Terms
    server.tool(
      'get_search_terms',
      'Get actual search queries that triggered your ads. Great for finding keyword opportunities or negatives.',
      {
        days:            z.number().default(30).describe('Lookback window in days'),
        campaign_filter: z.string().optional().describe('Filter by campaign name substring'),
        limit:           z.number().default(100).describe('Max number of search terms to return'),
        min_impressions: z.number().default(5).describe('Minimum impressions to include'),
      },
      async ({ days = 30, campaign_filter, limit = 100, min_impressions = 5 }) => {
        const { start, end } = dateRange(days)
        let where = `segments.date BETWEEN '${start}' AND '${end}'`
        if (campaign_filter) where += ` AND campaign.name LIKE '%${campaign_filter}%'`

        const rows = await gaql(`
          SELECT
            search_term_view.search_term,
            campaign.name,
            metrics.clicks,
            metrics.impressions,
            metrics.conversions,
            metrics.cost_micros
          FROM search_term_view
          WHERE ${where}
          ORDER BY metrics.impressions DESC
          LIMIT ${limit * 5}
        `)

        const map = new Map<string, { term: string; campaign: string; clicks: number; impr: number; conv: number; cost: number }>()
        for (const row of rows) {
          const term     = String(pick(row, 'searchTermView.searchTerm') ?? '')
          const campaign = String(pick(row, 'campaign.name')             ?? '')
          const cur      = map.get(term) ?? { term, campaign, clicks: 0, impr: 0, conv: 0, cost: 0 }
          cur.clicks += Number(pick(row, 'metrics.clicks')      ?? 0)
          cur.impr   += Number(pick(row, 'metrics.impressions') ?? 0)
          cur.conv   += Number(pick(row, 'metrics.conversions') ?? 0)
          cur.cost   += toInr(pick(row, 'metrics.costMicros'))
          map.set(term, cur)
        }

        const results = [...map.values()]
          .filter(t => t.impr >= min_impressions)
          .sort((a, b) => b.impr - a.impr)
          .slice(0, limit)

        if (!results.length) return { content: [{ type: 'text' as const, text: 'No search term data found for this period.' }] }

        const lines = results.map(t =>
          `• "${t.term}"\n  Impr ${t.impr} | Clicks ${t.clicks} | Conv ${fmt(t.conv, 1)} | Spend ₹${fmt(t.cost)}`
        )

        return { content: [{ type: 'text' as const, text: `Search Terms — last ${days} days (min ${min_impressions} impressions)\n${divider()}\n${lines.join('\n\n')}` }] }
      }
    )

    // Tool 4: Conversion Stats
    server.tool(
      'get_conversion_stats',
      'Get conversion breakdown by conversion action name. Shows which CTAs are converting.',
      {
        days: z.number().default(30).describe('Lookback window in days'),
      },
      async ({ days = 30 }) => {
        const { start, end } = dateRange(days)

        const rows = await gaql(`
          SELECT
            segments.conversion_action_name,
            metrics.conversions,
            metrics.all_conversions
          FROM campaign
          WHERE campaign.status = 'ENABLED'
            AND segments.date BETWEEN '${start}' AND '${end}'
          ORDER BY metrics.conversions DESC
        `)

        const map = new Map<string, { name: string; conv: number; allConv: number }>()
        for (const row of rows) {
          const name = String(pick(row, 'segments.conversionActionName') ?? 'Unknown')
          const cur  = map.get(name) ?? { name, conv: 0, allConv: 0 }
          cur.conv    += Number(pick(row, 'metrics.conversions')    ?? 0)
          cur.allConv += Number(pick(row, 'metrics.allConversions') ?? 0)
          map.set(name, cur)
        }

        const results = [...map.values()]
          .filter(a => a.conv > 0 || a.allConv > 0)
          .sort((a, b) => b.conv - a.conv)

        if (!results.length) return { content: [{ type: 'text' as const, text: 'No conversion data found for this period.' }] }

        const total = results.reduce((s, a) => s + a.conv, 0)
        const lines = results.map(a => {
          const pct = total > 0 ? ` (${fmt(a.conv / total * 100, 1)}%)` : ''
          return `• ${a.name}: ${fmt(a.conv, 1)} conversions${pct}`
        })

        return {
          content: [{
            type: 'text' as const,
            text: `Conversion Stats — last ${days} days\nTotal: ${fmt(total, 1)} conversions\n${divider()}\n${lines.join('\n')}`,
          }],
        }
      }
    )

    // Tool 5: Budget Pacing
    server.tool(
      'get_budget_pacing',
      "Check how campaigns are pacing against their daily budgets.",
      {},
      async () => {
        const { today } = dateRange(0)

        const rows = await gaql(`
          SELECT
            campaign.id,
            campaign.name,
            campaign_budget.amount_micros,
            metrics.cost_micros
          FROM campaign
          WHERE campaign.status = 'ENABLED'
            AND segments.date BETWEEN '${today}' AND '${today}'
          ORDER BY metrics.cost_micros DESC
        `)

        const map = new Map<string, { name: string; budget: number; spend: number }>()
        for (const row of rows) {
          const id     = String(pick(row, 'campaign.id') ?? 'x')
          const name   = String(pick(row, 'campaign.name') ?? '')
          const budget = toInr(pick(row, 'campaignBudget.amountMicros'))
          const cur    = map.get(id) ?? { name, budget, spend: 0 }
          cur.spend += toInr(pick(row, 'metrics.costMicros'))
          map.set(id, cur)
        }

        const results = [...map.values()].sort((a, b) => b.spend - a.spend)
        if (!results.length) return { content: [{ type: 'text' as const, text: `No spend data found for today (${today}).` }] }

        const lines = results.map(c => {
          const pct = c.budget > 0 ? `${fmt(c.spend / c.budget * 100, 1)}%` : 'N/A'
          return `• ${c.name}\n  Spend ₹${fmt(c.spend)} / Budget ₹${fmt(c.budget)} — Pacing ${pct}`
        })

        return { content: [{ type: 'text' as const, text: `Budget Pacing — ${today}\n${divider()}\n${lines.join('\n\n')}` }] }
      }
    )

    // Tool 6: Lookup GCLID
    // Constraints: click_view requires EXACTLY one day filter; segments.ad_network_type 
    // not click_view.ad_network_type; conversion metrics cannot be selected alongside click_view
    server.tool(
      'lookup_gclid',
      'Look up a specific gclid to check if the click was registered and get its campaign/ad group details.',
      {
        gclid: z.string().describe('The Google Click ID to look up'),
        days:  z.number().default(30).describe('Lookback window in days'),
      },
      async ({ gclid, days = 30 }) => {
        // click_view requires segments.date filter on EXACTLY one day per request.
        // We loop backwards from today until we find it or exhaust the window.
        for (let i = 0; i <= days; i++) {
          const day = new Date(Date.now() - i * 86_400_000).toISOString().slice(0, 10)
          
          const rows = await gaql(`
            SELECT
              click_view.gclid,
              campaign.name,
              ad_group.name,
              segments.date,
              segments.ad_network_type
            FROM click_view
            WHERE click_view.gclid = '${gclid}'
              AND segments.date BETWEEN '${day}' AND '${day}'
            LIMIT 1
          `)

          if (rows.length > 0) {
            const row      = rows[0]
            const campaign = String(pick(row, 'campaign.name')          ?? 'Unknown')
            const adGroup  = String(pick(row, 'adGroup.name')           ?? 'Unknown')
            const date     = String(pick(row, 'segments.date')          ?? 'Unknown')
            const network  = String(pick(row, 'segments.adNetworkType') ?? 'Unknown')

            return {
              content: [{
                type: 'text' as const,
                text: [
                  `GCLID Found ✓`,
                  divider(),
                  `GCLID:      ${gclid}`,
                  `Campaign:   ${campaign}`,
                  `Ad Group:   ${adGroup}`,
                  `Click Date: ${date}`,
                  `Network:    ${network}`,
                ].join('\n'),
              }],
            }
          }
        }

        return {
          content: [{
            type: 'text' as const,
            text: `GCLID not found: ${gclid}\n\nNot registered in the last ${days} days. It may be older, invalid, or the click was filtered by Google.`,
          }],
        }
      }
    )

  },
  {},
  {
    basePath:    '/api',
    maxDuration: 60,
    verboseLogs: true,
  }
)

export { handler as GET, handler as POST, handler as DELETE }
```

---

## Step 3 — Push to GitHub

```bash
npm install
git add .
git commit -m "feat: initial MCP server with 6 Google Ads tools"
git push
```

---

## Step 4 — Create Vercel Project

1. Go to [vercel.com](https://vercel.com) → `analytixlabs-team`
2. **Add New Project** → Import `alabs-mcp-server` from `dhaval-alabs` GitHub
3. Framework: **Next.js** (auto-detected)
4. Click **Deploy** (first deploy may fail — that's OK, env vars come next)

---

## Step 5 — Add Environment Variables

In Vercel → `alabs-mcp-server` → **Settings → Environment Variables**, add:

| Variable | Value | Source |
|---|---|---|
| `GOOGLE_ADS_CLIENT_ID` | Copy from careersuccess-legacy | |
| `GOOGLE_ADS_CLIENT_SECRET` | Copy from careersuccess-legacy | |
| `GOOGLE_ADS_REFRESH_TOKEN` | Copy from careersuccess-legacy | |
| `GOOGLE_ADS_DEVELOPER_TOKEN` | Copy from careersuccess-legacy | |
| `GOOGLE_ADS_CUSTOMER_ID` | `4064995850` | Optional (fallback built-in) |
| `GOOGLE_ADS_MCC_ID` | `8910137241` | Optional (fallback built-in) |

After saving env vars, trigger a **Redeploy**.

---

## Step 6 — Update Claude MCP Settings

In Claude.ai → Settings → Integrations, find the existing MCP entry and update the URL:

| | Old | New |
|---|---|---|
| URL | `https://careersuccess-legacy.vercel.app/api/mcp` | `https://alabs-mcp-server.vercel.app/api/mcp` |

---

## Step 7 — Verify

Confirm the server is live by visiting: `https://alabs-mcp-server.vercel.app/api/mcp`

You should see:
```json
{"jsonrpc":"2.0","error":{"code":-32000,"message":"Method not allowed."},"id":null}
```

This is correct — it means `mcp-handler` is running and SSE (GET) is disabled as expected.

Then open a fresh Claude chat and run `get_conversion_stats` — it should return real data.

---

## What NOT to do

- Do not add any pages, components, or UI to this project
- Do not add this MCP route to careersuccess-legacy (leave that code as-is)
- Do not change the Google Ads credentials after copying — both projects share the same Google Ads account
