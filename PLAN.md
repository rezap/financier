# Financier — initial plan

Goal: give an AI agent read access to my own Norwegian bank data (and later investments) so it can track, categorise and advise on spending. Write actions such as payments or trades come later, if at all, and always need my approval.

## 1. How to get the bank data (Norway, PSD2)

Norwegian banks must expose account data through PSD2 "Account Information Service" (AIS) APIs. Only licensed AISPs can call them directly. A private person can't get a licence, so we go through a licensed aggregator. The consumer authenticates with BankID at their own bank.

| Option | Norway coverage | Private/hobby use | Verdict |
|---|---|---|---|
| **Enable Banking** (Finnish, licensed AISP) | DNB, Sbanken, SpareBank 1 banks, other Norwegian banks | **Free "restricted production" mode**: you link your *own* accounts to your app, and only those accounts can be read. The ToS allows private personal use. | **Primary choice** |
| GoCardless Bank Account Data (ex-Nordigen) | Good | Was free, but **closed to new signups since July 2025** | Not available |
| Neonomics (Norwegian, Finanstilsynet-licensed) | Strong (~100 Nordic institutions) | B2B / sales-led, no self-serve hobby tier found | Fallback if we ever productise |
| Tink (Visa), Aiia/Nordic API Gateway (Mastercard) | Strong | Enterprise contracts | Not suitable for a personal project |
| Bank CSV export | All banks | Manual | Fallback / backfill of history |

**Decision:** start with Enable Banking in restricted production mode, and support CSV import as a fallback.

Practical constraints:
- PSD2 consent expires (max ~180 days), so the user re-authenticates with BankID about twice a year. The app must detect expiry and ask for re-consent.
- Banks limit unattended AIS calls (typically 4 per day per account). Sync on a schedule and cache everything locally, and never let the agent call the bank live.
- History depth varies by bank. The first sync may return only 90 days to 2 years, so backfill older data from CSV.

## 2. Investments
- **Nordnet** has an official API (`public.nordnet.no/api/2`, Ed25519 key auth), but its docs say it is **not onboarding new customers**. Check whether that has changed. Otherwise use CSV export of transactions and holdings.
- Bank-held funds and ASK accounts sometimes show up as accounts through PSD2, but usually only the cash balance.
- Plan: start read-only with CSV or manual import of holdings, and get prices from a public market-data source. **No automated trading** in scope.

## 3. Architecture (v0)

```
 BankID consent ──► Enable Banking API ──► sync worker (cron, ≤4x/day)
                                                │
 CSV imports (bank, Nordnet) ───────────────────┤
                                                ▼
                                      local SQLite (encrypted at rest)
                                     accounts · transactions · balances
                                     categories · rules · budgets · holdings
                                                │
                                                ▼
                                   MCP server (read-only tools)
                     list_accounts · query_transactions · spending_summary
                     categorise(tx, cat) · set_budget · get_holdings
                                                │
                                                ▼
                         Claude agent (Claude Code / Desktop / Agent SDK)
                     weekly review · anomaly alerts · budget coaching
```

- Language: Python (`httpx`, `pydantic`, `sqlite`), plus the official MCP Python SDK.
- Enable Banking auth: RS256-signed JWT using the app's private key. Flow: `POST /auth`, then the user does BankID in a browser, then the redirect returns a code, then `POST /sessions`, then `GET /accounts/{id}/transactions`.
- The agent talks to **our MCP server only**. It never sees bank credentials or keys, and it reads only from the local DB.

## 4. Milestones
1. **M0 – Access spike.** Register at Enable Banking, create the app and key pair, link my own DNB/SpareBank 1/etc. account, and fetch balances and transactions with a script.
2. **M1 – Sync + storage.** Idempotent sync into SQLite: dedupe by bank transaction ID, handle pending→booked transitions, track consent expiry. Add a CSV importer for history.
3. **M2 – Categorisation.** Rules first, such as merchant → category. Use the LLM for transactions no rule matches, and save confirmed results as new rules. Norwegian merchant strings (Vipps, "VISA VARE", etc.) need normalising.
4. **M3 – MCP server.** Read-only query tools plus a small set of write tools that only touch the local DB (categories, budgets, notes).
5. **M4 – Agent workflows.** A weekly spending review, subscription detection, unusual-transaction alerts, and progress against monthly budgets.
6. **M5 – Investments (read-only).** Holdings import, portfolio value and allocation, and savings-rate tracking.
7. **Later / optional.** PSD2 payment initiation (PIS), or trade suggestions, only with explicit human approval for each action.

## 5. Security & privacy
- Financial data is sensitive personal data. Keep it local and encrypt the DB file. Keep the Enable Banking private key outside the repo, in `.env` or the OS keychain, and gitignore all data.
- Transactions sent to the LLM leave the machine. Minimise this by sending aggregates or redacted fields where possible.
- Stay read-only by default. Any future write action (payment, trade) needs human confirmation.

## 6. Open questions for you
1. Which banks and accounts? Some Norwegian banks are only reachable through specific aggregators.
2. Where should this run: a laptop, a home server, or a small VPS? This affects the BankID redirect URL and the cron job.
3. Which investment platforms (Nordnet, DNB, Kron, Firi, etc.)?
4. How do you want to use it: chat (Claude Desktop with MCP), scheduled reports (email or Telegram), or both?

## 7. Network access needed for development
- `api.enablebanking.com`, `enablebanking.com` (API + docs)
- `public.nordnet.no`, `www.nordnet.no` (only if checking the Nordnet API)
- `pypi.org`, `files.pythonhosted.org` (Python packages)
