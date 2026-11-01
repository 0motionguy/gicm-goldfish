# @gicm/goldfish

Token and cost budgets for AI agents: daily, weekly and monthly limits with soft, throttle and hard thresholds, backed by SQLite.

Goldfish records the cost of each model call, compares spend with the budgets you set and tells you when to warn, slow down or stop. It is for developers running agents on metered APIs who want a hard stop before a bad day becomes a large bill. It also ships a pricing table that turns token counts into dollars.

Part of [gICM](https://github.com/0motionguy/gICM), a family of OpenClaw agent packages. API reference and examples: [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md).

## What it does

- **Budgets:** `BudgetManager(config, dbPath?)` takes zod-validated daily (reset time), weekly (reset day) and monthly (reset day 1 to 28) limits.
- **Thresholds:** soft at 70%, throttle at 90% and hard at 100% by default. `budget:soft`, `budget:throttle` and `budget:hard` fire when a period changes level, and `cost:recorded` fires on every record.
- **Record:** `recordCost(event)` stores model, provider, input, output, cache-read and cache-write tokens, cost, agent, session, task type and tier.
- **Check:** `getStatus(period)`, `getReport()`, `shouldThrottle()` and `shouldBlock()`. The two quick checks look at the daily period.
- **Price:** `calculateCost(model, tokens)` uses a static table (a February 2026 snapshot) covering Anthropic, OpenAI, Google, DeepSeek, xAI and Moonshot models; unknown models get a mid-tier estimate and a warning.
- **Query:** the cost database can sum spend by model and by agent.

## Example

```ts
import { BudgetManager, calculateCost } from "@gicm/goldfish";

const budget = new BudgetManager({
  daily: { amount: 10, resetAt: "00:00" },
  weekly: { amount: 50, resetDay: "monday" },
  monthly: { amount: 200, resetDay: 1 },
  thresholds: { soft: 0.7, throttle: 0.9, hard: 1 },
});
budget.on("budget:hard", (status) => console.warn("budget hit", status.spent));

const cost = calculateCost("claude-sonnet-4-5", { input: 100_000, output: 20_000 });
budget.recordCost({
  timestamp: new Date().toISOString(), model: "claude-sonnet-4-5", provider: "anthropic",
  inputTokens: 100_000, outputTokens: 20_000, cacheReadTokens: 0, cacheWriteTokens: 0,
  cost, agentId: "agent-1", sessionId: "s-1", taskType: "chat", tier: 1,
});
if (budget.shouldBlock()) { /* stop calling the model */ }
```

## How it's built

TypeScript (ESM) bundled with tsup and tested with vitest. Spend is stored with `better-sqlite3` (WAL mode, default path under the user's home `.openclaw` folder); if the native module cannot load, an in-memory store takes over. zod validates the budget config.

## Repository layout

- `src/`: `goldfish` (budget logic), `db` (SQLite and fallback), `pricing`, `types`
- `src/__tests__/`: vitest suite (18 test cases)
- `skill/`: `SKILL.md` (the OpenClaw skill definition) and `references/pricing.md`

## Run it locally

```bash
pnpm install
pnpm build   # tsup -> dist/
pnpm test    # vitest (add --run for a single pass)
```

## Status

Beta. Version 1.0.0, a single extracted release (February 2026) with a test suite. The `perSession` and `perAgent` limits are accepted in the config but not enforced yet, and the pricing table needs updating as provider prices change.
