# Grid Runners — Backend Implementation Guide

> **For:** You (the backend person)  
> **Written in:** Simple words, no BS  
> **What this is:** A step-by-step plan to connect real backends to the game your friend built

---

## First, Let Me Explain What's Going On

Your friend built a complete game. It works right now — you can run it, play it, trade items, do missions. But here's the catch: **everything is fake under the hood.** 

- Players? Fake guest accounts (no real login)
- Saved data? Gone when you close the browser (it's in memory)
- Money/blockchain? Fake Solana with fake transactions
- Cache/speed? Just a JavaScript `Map()` object

**Your job:** Replace the fake stuff with real stuff. The game code stays the same — you only fill in the "adapter" files your friend left empty for you.

Think of it like this: your friend built a car with a working steering wheel, seats, and dashboard. But instead of a real engine, there's a guy pedaling underneath. You're swapping in the real engine.

---

## The 3 Services You Need to Set Up

| Service | What It Is (Simple) | Why the Game Needs It | Cost |
|---------|--------------------|-----------------------|------|
| **Supabase** | A database + login system (like Firebase but uses Postgres) | To save player data, items, trades, etc. permanently. Right now everything is lost when the server restarts | Free tier available |
| **Upstash Redis** | A super-fast temporary storage (like a notepad that auto-erases) | To stop players from spamming actions (rate limiting), track who's online, and speed things up | Free tier available |
| **Helius** | A door to the Solana blockchain | To mint real NFTs, do real crypto transactions, real marketplace escrow | Free tier (devnet is free) |

### "What is Redis?" (Since You Asked)

Redis is like a **super-fast sticky note board**. You write something on it, it's instantly available, and it disappears after a set time. The game uses it for:

- **Rate limiting:** "This player already collected salvage 2 seconds ago, don't let them again" 
- **Presence:** "Which players are online right now?"
- **Cooldowns:** "This player just did a courier run, they need to wait 15 seconds"
- **Speed:** Instead of asking the database every time "show me active marketplace listings," you ask Redis (way faster)

**Upstash** is just a company that hosts Redis for you, and you talk to it over regular HTTP (no special setup needed). Think of it as "Redis as a service."

---

## Phase Overview

```
Phase 0: Web Restructuring & Hybrid Backend  ← Frontend & routing, ~1-2 days
Phase 1: Create Accounts & Get Keys          ← 30 minutes, no code
Phase 2: Supabase (Database + Auth)          ← The biggest piece, ~2-3 days  
Phase 3: Upstash Redis (Cache)               ← Easiest, ~2-4 hours
Phase 4: Helius Solana (Blockchain)           ← Hardest, ~3-5 days
```

> [!TIP]
> **Do them in this order.** Phase 0 handles new page routing and hybrid auth. Phase 2 (Supabase) is the persistence foundation. Phase 3 (Redis) makes it fast. Phase 4 (Helius) adds real blockchain.

---

# Phase 0: Web Restructuring & Hybrid Backend

**What happens after this phase:** The website will have a new pixelated GTA 1 themed homepage, a dedicated auth page, a whitepaper page, and the main game page. The backend will support a hybrid storage model where guests are run entirely in memory and authenticated players save to Supabase.

---

## Task 0.1 — Frontend Routing & Pages

1. **Move Game Entry Point:** Move `apps/web/app/page.tsx` to `apps/web/app/game/page.tsx`.
2. **New Homepage:** Create a new `apps/web/app/page.tsx` that has a pixelated GTA 1 theme using assets from `apps/web/public/assets/placeholder/`. It should have a scrolling effect, a "Play Now" button (routing to `/game`), and a "Connect Wallet" button (routing to `/auth`). Include a link to the whitepaper.
3. **Auth Page:** Create `apps/web/app/auth/page.tsx` where users can input their name, connect their wallet, and then be redirected to `/game` as authenticated users. **We will build a custom Solana adapter that ONLY connects to Phantom Wallet.**
4. **Whitepaper Page:** Create `apps/web/app/whitepaper/page.tsx` to hold the roadmap and game data.

## Task 0.2 — Hybrid Backend Storage Model

Since we need to support in-memory guests and Supabase-authenticated players in the *same* game room simultaneously:
1. Create `HybridPlayerRepository`, `HybridInventoryRepository`, etc., inside `packages/adapters/`.
2. These hybrid repositories should check the `playerId` or session type on every read/write. If the user is a guest, they route the call to `InMemory*Repository`. If authenticated, they route to `Supabase*Repository`.
3. **Block P2P Trading between Guests and Auth Players:** Because a guest's items vanish when they refresh, any items they trade to an authenticated player (or vice versa) could lead to permanent loss or imbalance. Add logic in `TradeService` (or `HybridTradeRepository`) to strictly prevent guests from trading with authenticated players.
4. Update `createGameServices` in `services.ts` to instantiate and use these Hybrid repositories.

---

# Phase 1: Create Accounts & Get Keys

**What happens after this phase:** You'll have all the API keys and URLs you need. No code yet.

---

## Task 1.1 — Create a Supabase Project

### What is this?
Supabase gives you a Postgres database (where player data lives forever) and a login system. You're creating a free project to get your database URL and API keys.

### What happens when I do this?
You get 3 values you'll paste into your `.env` file: `SUPABASE_URL`, `SUPABASE_ANON_KEY`, and `SUPABASE_SERVICE_ROLE_KEY`. The game will then save player profiles, items, trades, and marketplace listings to a real database instead of RAM.

### How to do it:

1. Go to [https://supabase.com](https://supabase.com) and sign up (GitHub login works)
2. Click **"New Project"**
3. Fill in:
   - **Name:** `grid-runners` (or whatever you want)
   - **Database Password:** Pick a strong password and **save it somewhere** — you'll need it later
   - **Region:** Pick the one closest to your players
   - **Plan:** Free tier is fine to start
4. Wait ~2 minutes for it to spin up
5. Go to **Settings → API** (left sidebar)
6. Copy these 3 values:

```
SUPABASE_URL=https://xxxxx.supabase.co        ← "Project URL"
SUPABASE_ANON_KEY=eyJhbG...                    ← "anon public" key  
SUPABASE_SERVICE_ROLE_KEY=eyJhbG...            ← "service_role" key (⚠️ KEEP SECRET)
```

> [!WARNING]
> The `SERVICE_ROLE_KEY` bypasses all security rules. **Never** put it in your frontend code. It only goes on the game server.

### Alternative options:
| Option | Pros | Cons |
|--------|------|------|
| **Supabase (recommended)** | Free tier, your friend designed for it, schema ready | Vendor lock-in |
| **Self-hosted Postgres** | Full control, no vendor | You manage everything yourself |
| **PlanetScale (MySQL)** | Great free tier | Schema needs rewriting (it's written for Postgres) |
| **Neon (Postgres)** | Serverless Postgres, generous free | Need to adapt Supabase auth part separately |

---

## Task 1.2 — Create the Database Tables

### What is this?
Your friend already wrote the database structure (the tables, columns, rules) in a file called `schema.sql`. You just need to run it in your new Supabase project.

### What happens when I do this?
Your database gets 10 tables: `users`, `player_profiles`, `item_instances`, `marketplace_listings`, `trade_sessions`, `economy_events`, etc. These are where all the game data will live.

### How to do it:

1. In Supabase dashboard, click **"SQL Editor"** (left sidebar)
2. Click **"New Query"**
3. Open the file [schema.sql](file:///d:/gta1-main/db/schema.sql) from your project
4. Copy the entire contents and paste it into the SQL editor
5. Click **"Run"** (the play button)
6. You should see "Success. No rows returned" — that means the tables were created

### Verify it worked:
- Go to **"Table Editor"** in Supabase  
- You should see all 10+ tables listed: `users`, `player_profiles`, `item_definitions`, `item_instances`, `inventory_locks`, `marketplace_listings`, `trade_sessions`, `trade_offers`, `economy_events`, `job_runs`, `active_crafts`, `crafting_recipes`

---

## Task 1.3 — Create an Upstash Redis Instance

### What is this?
You're creating a Redis database (the fast sticky-note board I explained above). Upstash hosts it for you and gives you a REST API (so you just make HTTP calls — no special Redis drivers needed).

### What happens when I do this?
You get 2 values: `UPSTASH_REDIS_REST_URL` and `UPSTASH_REDIS_REST_TOKEN`. The game will use this for rate limiting (stopping spam), cooldowns, and caching.

### How to do it:

1. Go to [https://upstash.com](https://upstash.com) and sign up
2. Click **"Create Database"**
3. Fill in:
   - **Name:** `grid-runners-cache`
   - **Region:** Same region as your Supabase project (for speed)
   - **Type:** Regional (not Global, unless you need multi-region)
4. After creation, go to the **"REST API"** tab
5. Copy these 2 values:

```
UPSTASH_REDIS_REST_URL=https://xxxxx.upstash.io   ← "UPSTASH_REDIS_REST_URL"
UPSTASH_REDIS_REST_TOKEN=AXxx...                   ← "UPSTASH_REDIS_REST_TOKEN"
```

### Alternative options:
| Option | Pros | Cons |
|--------|------|------|
| **Upstash (recommended)** | Free tier (10K commands/day), REST API (easy), serverless | Limited free tier |
| **Redis Cloud** | More traditional Redis | Needs Redis driver, more setup |
| **Self-hosted Redis** | Full control, unlimited | You manage the server |
| **Vercel KV** | Integrated if using Vercel | Built on Upstash anyway, Vercel-locked |

---

## Task 1.4 — Get Helius API Key Ready

### What is this?
Helius is a service that lets you talk to the Solana blockchain easily. You said you already have the API key — great! You'll also need to decide which Solana network to start on.

### What happens when I do this?
The game will be able to mint real NFTs (items as compressed NFTs), do real marketplace escrow, and settle trades on the Solana blockchain.

### How to do it:

1. You already have your Helius API key ✅
2. **Start on devnet** (fake money, safe to test):

```
HELIUS_API_KEY=your-key-here
SOLANA_RPC_URL=https://devnet.helius-rpc.com/?api-key=your-key-here
SOLANA_NETWORK=devnet
TREASURY_WALLET=your-devnet-wallet-address
```

3. To get a devnet wallet: install [Phantom wallet](https://phantom.app/), switch to devnet in settings, copy your address
4. Get free devnet SOL: go to [https://faucet.solana.com](https://faucet.solana.com) and airdrop some

> [!NOTE]
> **Don't use mainnet yet.** Build everything on devnet first. When you're ready, just change `SOLANA_NETWORK=mainnet-beta` and swap the RPC URL.

---

## Task 1.5 — Set Up Your `.env` File

### What is this?
You're creating the file that holds all your secrets (API keys, URLs). The game reads this file to know which real services to use instead of the fake ones.

### How to do it:

1. Copy `.env.example` to `.env` in the project root:

```bash
cp .env.example .env
```

2. Open `.env` and fill in the values you collected. **For now, only turn on what you're ready for:**

```bash
# ---- Start with these (Phase 2) ----
DB_PROVIDER=supabase
AUTH_PROVIDER=supabase
SUPABASE_URL=https://xxxxx.supabase.co
SUPABASE_ANON_KEY=eyJhbG...
SUPABASE_SERVICE_ROLE_KEY=eyJhbG...

# ---- Add these when ready (Phase 3) ----
# CACHE_PROVIDER=redis
# UPSTASH_REDIS_REST_URL=https://xxxxx.upstash.io
# UPSTASH_REDIS_REST_TOKEN=AXxx...

# ---- Add these when ready (Phase 4) ----
# CHAIN_PROVIDER=helius
# HELIUS_API_KEY=your-key
# SOLANA_RPC_URL=https://devnet.helius-rpc.com/?api-key=your-key
# SOLANA_NETWORK=devnet
# TREASURY_WALLET=your-wallet-address

# ---- Turn these OFF for production ----
NEXT_PUBLIC_DEV_FAST_GRIND=false
NEXT_PUBLIC_ENABLE_ADMIN=false
```

3. **Keep lines commented out (with `#`) until you've implemented that adapter.** Otherwise the game will try to use the real service with the "not implemented" stub and crash.

---

# Phase 2: Supabase Backend (Database + Auth)

**What happens after this phase:** Player data, items, marketplace listings, and trades are saved permanently in a real database. Players can close their browser and come back to find everything still there.

**Time estimate:** 2-3 days

> [!IMPORTANT]
> This is the biggest phase. Your friend left 6 empty classes in [supabase.ts](file:///d:/gta1-main/packages/adapters/src/stubs/supabase.ts). You need to fill each one with real Supabase queries. The in-memory versions at [repositories.ts](file:///d:/gta1-main/packages/adapters/src/memory/repositories.ts) show you exactly what each method should do.

---

## Task 2.1 — Install the Supabase Client Library

### What is this?
You need to add the Supabase JavaScript library to the project so your code can talk to Supabase.

### What happens when I do this?
Your TypeScript code gets access to `createClient()` from Supabase, which lets you run database queries and verify login tokens.

### Prompt (what to tell the AI / do yourself):

```
Install the @supabase/supabase-js package in the adapters package:

cd d:\gta1-main
pnpm add @supabase/supabase-js --filter @grid/adapters
```

After installing, at the top of `packages/adapters/src/stubs/supabase.ts`, add:

```typescript
import { createClient, type SupabaseClient } from "@supabase/supabase-js";
```

And add a helper to create the client inside the file:

```typescript
function makeClient(config: SupabaseConfig): SupabaseClient {
  return createClient(config.url, config.serviceRoleKey);
}
```

---

## Task 2.2 — Implement SupabaseAuthProvider

### What is this?
This is the login system. Right now, players get a random "guest" ID. With this, they'll log in through Supabase (email, Google, wallet, etc.) and get a real persistent identity.

### What happens when I do this?
Players can create accounts, log in, and their identity persists across sessions. Their wallet address gets linked to their account.

### How it works:
1. The client sends a Supabase JWT token as the "credential"
2. Your code verifies the JWT with Supabase
3. You look up or create the user in the `users` table
4. You return their identity (userId, displayName, walletAddress)

### Prompt:

```
Implement the SupabaseAuthProvider.authenticate() method in 
packages/adapters/src/stubs/supabase.ts

It should:
1. Use supabase.auth.getUser(credential) to verify the JWT token
2. If invalid, throw an error
3. Query the 'users' table for this user.
4. If user doesn't exist, INSERT a new row into 'users' with:
   - id = the user's wallet address (DO NOT generate a randomized ID)
   - username = displayName or email or 'Runner-XXXX'
   - wallet_address = the user's wallet address
5. Return an AuthIdentity object:
   {
     userId: user.id, // This is the wallet address
     displayName: user.username, 
     walletAddress: user.wallet_address,
     isGuest: false
   }

Reference the MockAuthProvider in packages/adapters/src/mock/auth.ts to see
the return type shape.
```

### Alternative options for auth:
| Option | Pros | Cons |
|--------|------|------|
| **Supabase Auth (recommended)** | Built-in, supports email/Google/wallet | Tied to Supabase |
| **Privy** | Great for Web3 (email → wallet) | Extra service, extra cost |
| **NextAuth.js** | Flexible, many providers | Need custom session handling |
| **Keep guest auth** | No work needed | No persistence, no real identity |

---

## Task 2.3 — Implement SupabasePlayerRepository

### What is this?
This saves and loads player profiles — their level, XP, cash, reputation, what vehicle they're driving, etc.

### What happens when I do this?
Player progress is saved permanently. They can close the game and come back with all their stats intact.

### Prompt:

```
Implement all 3 methods of SupabasePlayerRepository in 
packages/adapters/src/stubs/supabase.ts

Study the InMemoryPlayerRepository in packages/adapters/src/memory/repositories.ts
and the player_profiles table in db/schema.sql.

Methods:

1. get(id: PlayerId) → Promise<PlayerProfile | undefined>
   - SELECT * FROM player_profiles WHERE user_id = id
   - Map database snake_case columns to TypeScript camelCase PlayerProfile
   - Return undefined if not found

2. upsert(profile: PlayerProfile) → Promise<void>
   - INSERT INTO player_profiles (...) VALUES (...) 
     ON CONFLICT (user_id) DO UPDATE SET ... WHERE version = profile.version
   - Increment the version column by 1 on each update
   - If the version doesn't match (someone else updated), throw an error
     (this is "optimistic concurrency" — prevents two servers writing at once)

3. all() → Promise<PlayerProfile[]>
   - SELECT * FROM player_profiles
   - Map all rows to PlayerProfile objects

Key mapping (database → TypeScript):
  user_id → playerId
  street_cash → streetCash  
  grid_coin → gridCoin
  equipped_cosmetics → equippedCosmetics (JSON column)
  current_vehicle_instance_id → currentVehicleInstanceId
```

---

## Task 2.4 — Implement SupabaseInventoryRepository

### What is this?
This saves and loads items that players own — their vehicles, materials (scrap, fragments), blueprints, weapons, etc.

### What happens when I do this?
Items are saved permanently. When a player crafts the Neon Wraith, it stays in their inventory forever (not just until they close the tab).

### Prompt:

```
Implement all 4 methods of SupabaseInventoryRepository in 
packages/adapters/src/stubs/supabase.ts

Study the InMemoryInventoryRepository in packages/adapters/src/memory/repositories.ts
and the item_instances + inventory_locks tables in db/schema.sql.

Methods:

1. get(id: ItemInstanceId) → Promise<ItemInstance | undefined>
   - SELECT * FROM item_instances WHERE id = $id
   - Map to ItemInstance type (see packages/shared/src/types.ts)

2. itemsByOwner(ownerUserId: PlayerId) → Promise<ItemInstance[]>
   - SELECT * FROM item_instances WHERE owner_user_id = $ownerUserId
   - Return all items owned by this player

3. upsert(item: ItemInstance) → Promise<void>
   - INSERT INTO item_instances (...) ON CONFLICT (id) DO UPDATE SET ...
     WHERE version = item.version
   - IMPORTANT: When lock_status changes, also INSERT/DELETE from 
     inventory_locks table to keep them in sync
   - Increment version on update

4. delete(id: ItemInstanceId) → Promise<void>
   - DELETE FROM item_instances WHERE id = $id
   - Also deletes from inventory_locks (cascade should handle this)

Key columns to map:
  owner_user_id, item_definition_id, quantity, durability, rarity_roll,
  stats_json (JSONB), cosmetic_json (JSONB), asset_mint_address, 
  chain_status, lock_status, lock_ref, version
```

> [!WARNING]
> **The lock system is critical.** When an item is listed on the marketplace, its `lock_status` becomes `'marketplace'` and `lock_ref` stores the listing ID. If you mess this up, items can be duplicated or stolen. Study the trading package tests carefully.

---

## Task 2.5 — Implement SupabaseMarketplaceRepository

### What is this?
This saves marketplace listings — when a player puts an item up for sale, the listing is stored here.

### What happens when I do this?
Marketplace listings survive server restarts. All players across all sessions see the same listings.

### Prompt:

```
Implement all 4 methods of SupabaseMarketplaceRepository in 
packages/adapters/src/stubs/supabase.ts

Study InMemoryMarketplaceRepository in packages/adapters/src/memory/repositories.ts
and marketplace_listings table in db/schema.sql.

Methods:

1. get(id: string) → Promise<MarketplaceListing | undefined>
   - SELECT * FROM marketplace_listings WHERE id = $id

2. active() → Promise<MarketplaceListing[]>  
   - SELECT * FROM marketplace_listings WHERE status = 'active'
   - These are the ones shown to buyers

3. all() → Promise<MarketplaceListing[]>
   - SELECT * FROM marketplace_listings (all statuses)

4. upsert(listing: MarketplaceListing) → Promise<void>
   - INSERT ... ON CONFLICT (id) DO UPDATE SET ...
   
Key columns: seller_user_id, item_instance_id, seller_name, 
  item_snapshot (JSONB), price_amount, price_currency, status,
  escrow_tx_signature, settlement_tx_signature, expires_at
```

---

## Task 2.6 — Implement SupabaseTradeRepository

### What is this?
This saves P2P trade sessions — when two players walk up to each other and trade items.

### What happens when I do this?
Trades are tracked in the database. If a server crashes mid-trade, the system can detect and clean up stuck trades.

### Prompt:

```
Implement all 3 methods of SupabaseTradeRepository in 
packages/adapters/src/stubs/supabase.ts

Study InMemoryTradeRepository in packages/adapters/src/memory/repositories.ts
and trade_sessions + trade_offers tables in db/schema.sql.

Methods:

1. get(id: string) → Promise<TradeSession | undefined>
   - SELECT from trade_sessions JOIN trade_offers
   - Build the full TradeSession object with both players' offers

2. upsert(session: TradeSession) → Promise<void>
   - UPSERT trade_sessions row
   - UPSERT both trade_offers rows (initiator + counterparty)
   - This should ideally be in a transaction (both writes succeed or both fail)

3. activeForUser(userId: PlayerId) → Promise<TradeSession | undefined>
   - SELECT from trade_sessions WHERE status IN ('open','confirmed_by_one',
     'confirmed_by_both','executing') AND (initiator = userId OR counterparty = userId)
   - A player can only have one active trade at a time
```

---

## Task 2.7 — Implement SupabaseEconomyLedgerRepository

### What is this?
This is the audit log. Every time something economic happens (player earns cash, buys an item, crafts something), an event is logged here with a unique key so it can never be duplicated.

### What happens when I do this?
You get a complete history of every economic action in the game. If something goes wrong, you can trace exactly what happened. The idempotency keys prevent double-spending bugs.

### Prompt:

```
Implement all 4 methods of SupabaseEconomyLedgerRepository in 
packages/adapters/src/stubs/supabase.ts

Study InMemoryEconomyLedgerRepository in packages/adapters/src/memory/repositories.ts
and economy_events table in db/schema.sql.

Methods:

1. append(event: EconomyEvent) → Promise<void>
   - INSERT INTO economy_events (...) VALUES (...)
     ON CONFLICT (idempotency_key) DO NOTHING
   - ⚠️ The ON CONFLICT DO NOTHING is CRITICAL — it's what makes 
     replaying events safe (if the same event is sent twice, 
     the second one is silently ignored)

2. byIdempotencyKey(key: string) → Promise<EconomyEvent | undefined>
   - SELECT * FROM economy_events WHERE idempotency_key = $key

3. byUser(userId: PlayerId, limit?: number) → Promise<EconomyEvent[]>
   - SELECT * FROM economy_events WHERE user_id = $userId 
     ORDER BY created_at DESC LIMIT $limit

4. all() → Promise<EconomyEvent[]>
   - SELECT * FROM economy_events ORDER BY created_at DESC
```

---

## Task 2.8 — Wire Repositories Through the Registry

### What is this?
Right now, the game always creates `InMemory*` repositories (the fake ones). You need to make it choose between in-memory and Supabase based on the `DB_PROVIDER` env var.

### What happens when I do this?
When you set `DB_PROVIDER=supabase` in your `.env`, the game automatically uses your new Supabase code. Remove it, and it falls back to the in-memory version. No other code changes needed.

### Prompt:

```
Update packages/game-core/src/services.ts (the createGameServices function).

Right now it hard-codes InMemory* repositories. Change it so:
- If DB_PROVIDER=supabase (check process.env or pass in config), 
  create Supabase* repositories instead
- If DB_PROVIDER=memory (or not set), keep using InMemory* repos

You'll also need to update the registry.ts to export createRepositories() 
function, similar to how createAuthProvider() works.

The key idea: the GameRoom and all trading services don't care which 
repository they're using — they just call .get(), .upsert(), etc. 
The adapter pattern means you're swapping the engine without touching 
the steering wheel.
```

---

## Task 2.9 — Test Phase 2

### What is this?
Making sure your Supabase implementation actually works before moving on.

### How to do it:

```bash
# 1. Make sure the existing tests still pass (they use in-memory)
pnpm test

# 2. Set your env vars
DB_PROVIDER=supabase
AUTH_PROVIDER=supabase
# (plus your Supabase keys)

# 3. Start the game
pnpm dev

# 4. Open http://localhost:3000
# 5. Play for a few minutes — do a courier run, collect salvage, craft something
# 6. Close the browser tab
# 7. Open it again — your stuff should still be there!

# 8. Check Supabase dashboard — you should see rows in:
#    - users
#    - player_profiles  
#    - item_instances
#    - economy_events
```

### What to watch for:
- ❌ "Not implemented" errors → you missed a method
- ❌ "version mismatch" errors → your optimistic concurrency logic has a bug  
- ❌ Items disappearing → your upsert is deleting instead of updating
- ❌ Duplicate items → your lock system isn't working right
- ✅ Data shows up in Supabase dashboard → you're good!

---

## Task 2.10 — Supabase Implementation SQL Queries

When implementing the Supabase adapters, you will use `@supabase/supabase-js`, which translates to these precise queries. Note how optimistic concurrency works (`version` checks):

**Users / Auth (`SupabaseAuthProvider`)**:
```sql
-- Authenticating a user (Wallet Address as ID)
INSERT INTO users (id, wallet_address, username)
VALUES ('<wallet_address>', '<wallet_address>', '<displayName>')
ON CONFLICT (id) DO NOTHING;
```

**Player Profiles (`SupabasePlayerRepository`)**:
```sql
-- Get Profile
SELECT * FROM player_profiles WHERE user_id = '<wallet_address>';

-- Upsert Profile (Optimistic Concurrency)
INSERT INTO player_profiles (user_id, level, xp, street_cash, reputation, heat, grid_coin, equipped_cosmetics, current_vehicle_instance_id, version)
VALUES (...)
ON CONFLICT (user_id) DO UPDATE SET 
  level = EXCLUDED.level, 
  xp = EXCLUDED.xp,
  ...
  version = player_profiles.version + 1
WHERE player_profiles.version = <expected_version>;
```

**Inventory (`SupabaseInventoryRepository`)**:
```sql
-- Get Items by Owner
SELECT * FROM item_instances WHERE owner_user_id = '<wallet_address>';

-- Upsert Item (Optimistic Concurrency)
INSERT INTO item_instances (id, owner_user_id, item_definition_id, quantity, chain_status, lock_status, lock_ref, version, ...)
VALUES (...)
ON CONFLICT (id) DO UPDATE SET
  owner_user_id = EXCLUDED.owner_user_id,
  quantity = EXCLUDED.quantity,
  lock_status = EXCLUDED.lock_status,
  lock_ref = EXCLUDED.lock_ref,
  version = item_instances.version + 1
WHERE item_instances.version = <expected_version>;
```

**Marketplace (`SupabaseMarketplaceRepository`)**:
```sql
-- Upsert Listing
INSERT INTO marketplace_listings (id, seller_user_id, item_instance_id, price_amount, price_currency, status, expires_at)
VALUES (...)
ON CONFLICT (id) DO UPDATE SET
  status = EXCLUDED.status,
  escrow_tx_signature = EXCLUDED.escrow_tx_signature,
  settlement_tx_signature = EXCLUDED.settlement_tx_signature;
```

**Economy Ledger (`SupabaseEconomyLedgerRepository`)**:
```sql
-- Append Event (Idempotent)
INSERT INTO economy_events (user_id, event_type, idempotency_key, amount, currency)
VALUES (...)
ON CONFLICT (idempotency_key) DO NOTHING;
```

---

# Phase 3: Upstash Redis Cache

**What happens after this phase:** The game has rate limiting (players can't spam actions), cooldowns work across server restarts, and frequently-read data is cached for speed.

**Time estimate:** 2-4 hours (seriously, this is the easy one)

---

## Task 3.1 — Install the Upstash Redis Client

### What is this?
Adding the Upstash Redis library so your code can talk to Redis over HTTP.

### Prompt:

```
Install the @upstash/redis package in the adapters package:

cd d:\gta1-main
pnpm add @upstash/redis --filter @grid/adapters
```

---

## Task 3.2 — Implement UpstashRedisCacheProvider

### What is this?
You're filling in 4 simple methods: get a value, set a value (with optional expiry), delete a value, and increment a counter. That's it. Seriously.

### What happens when I do this?
- **Rate limiting works:** Players can't collect salvage every 0.1 seconds
- **Cooldowns survive restarts:** If a player has a 15-second courier cooldown and the server restarts, the cooldown is still there
- **Caching:** Frequently-read data loads faster

### Prompt:

```
Implement all 4 methods of UpstashRedisCacheProvider in 
packages/adapters/src/stubs/upstash.ts

Study the NoopCacheProvider in packages/adapters/src/mock/cache.ts — 
your implementation does the same thing but talks to real Redis.

Add this import at the top:
  import { Redis } from "@upstash/redis";

Add a Redis client in the constructor:
  private redis: Redis;
  constructor(private config: UpstashConfig) {
    this.redis = new Redis({
      url: config.restUrl,
      token: config.restToken,
    });
  }

Methods:

1. get(key: string) → Promise<string | null>
   - return await this.redis.get<string>(key);

2. set(key: string, value: string, ttlSeconds?: number) → Promise<void>
   - if (ttlSeconds) await this.redis.set(key, value, { ex: ttlSeconds });
   - else await this.redis.set(key, value);

3. del(key: string) → Promise<void>
   - await this.redis.del(key);

4. incr(key: string, ttlSeconds?: number) → Promise<number>
   - const val = await this.redis.incr(key);
   - if (ttlSeconds && val === 1) await this.redis.expire(key, ttlSeconds);
   - return val;
   (incr creates the key if it doesn't exist and sets it to 1)
```

> [!TIP]
> This is literally ~20 lines of real code. The `@upstash/redis` library does all the heavy lifting.

### Alternative options:
| Option | Pros | Cons |
|--------|------|------|
| **@upstash/redis (recommended)** | REST API, works everywhere (even serverless) | Slightly slower than direct Redis |
| **ioredis** | Fastest, most features | Needs persistent connection (no serverless) |
| **redis (node-redis)** | Official client | Same connection issue as ioredis |

---

## Task 3.3 — Test Phase 3

### How to do it:

```bash
# 1. Uncomment the Redis env vars in your .env:
CACHE_PROVIDER=redis
UPSTASH_REDIS_REST_URL=https://xxxxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=AXxx...

# 2. Start the game
pnpm dev

# 3. Play — collect salvage rapidly, try to spam actions
# 4. Check Upstash dashboard — you should see keys appearing
#    (rate limit counters, cooldown keys, etc.)
```

---

# Phase 4: Helius Solana Blockchain

**What happens after this phase:** Items can be real NFTs on Solana, marketplace trades use real escrow, and players can truly own their in-game items on the blockchain.

**Time estimate:** 3-5 days (this is the complex one)

> [!CAUTION]
> **Start on devnet!** Use fake SOL, fake transactions. Only move to mainnet when everything is battle-tested. A bug here can lose real money.

---

## Task 4.1 — Install Solana Dependencies

### What is this?
Adding the Solana libraries and Helius SDK so your code can interact with the blockchain.

### Prompt:

```
Install Solana dependencies in the adapters package:

cd d:\gta1-main
pnpm add @solana/web3.js @metaplex-foundation/mpl-bubblegum helius-sdk --filter @grid/adapters
```

---

## Task 4.2 — Implement Basic Chain Methods (Start Here)

### What is this?
Start with the simpler methods that don't involve escrow or atomic swaps. Get these working first, then build up to the complex stuff.

### Prompt (Round 1 — wallet & balance):

```
Implement these 4 methods in HeliusSolanaChainProvider 
(packages/adapters/src/stubs/helius.ts):

1. createWallet(seed: string)
   - In production, wallets come from the player's browser wallet 
     (Phantom, Backpack, etc. via Solana Wallet Adapter)
   - This method should just validate/derive, not create real wallets
   - For now: return { address: seed } (the seed IS the wallet address 
     passed from the client)

2. getBalance(walletAddress: string, currency: Currency)
   - Use Helius RPC to get SOL balance: 
     connection.getBalance(new PublicKey(walletAddress))
   - For GRID/SPL tokens: use connection.getTokenAccountsByOwner()
   - Return the balance as a number

3. getWalletAssets(walletAddress: string)
   - Use Helius DAS API: POST to your RPC URL with method 
     "getAssetsByOwner" and the wallet address
   - Map results to ChainAsset[] (mintAddress, name, etc.)
   - This shows all NFTs a player owns

4. getTransactionStatus(signature: string)  
   - Use connection.getSignatureStatuses([signature])
   - Map to TxStatus: "confirmed" | "pending" | "failed"
```

---

## Task 4.3 — Implement Minting

### What is this?
When a player crafts a rare item (like the Neon Wraith), it gets minted as a real compressed NFT (cNFT) on Solana. Compressed NFTs are super cheap (~$0.0001 each) because they use Merkle trees instead of full accounts.

### What happens when I do this?
Crafted mythic items become real blockchain assets that players truly own. They can see them in their Phantom wallet, on Magic Eden, etc.

### Prompt:

```
Implement mintItemAsset(item: ItemInstance) in HeliusSolanaChainProvider.

This should:
1. Build metadata for the NFT from the item's definition 
   (name, image URL, attributes like rarity)
2. Use Metaplex Bubblegum to mint a compressed NFT (cNFT):
   - You need a Merkle tree (create one beforehand, or use a shared one)
   - The tree account holds many NFTs efficiently
3. Return { signature, confirmed: true, mintAddress }
4. The mintAddress gets stored on the item in the database 
   (item_instances.asset_mint_address)

Alternative (simpler): Use Metaplex Core instead of Bubblegum:
- Metaplex Core assets are simpler to create
- They cost slightly more but the API is much cleaner
- Use @metaplex-foundation/mpl-core

Start simple:
- Mint on devnet with a test tree
- Hardcode the tree address for now
- Verify mints show up in the player's wallet via getAssetsByOwner
```

---

## Task 4.4 — Implement Marketplace Escrow

### What is this?
When a player lists an item for sale, the NFT needs to be held in escrow (locked in a program-owned account) so the seller can't take it back while someone is buying it. When someone buys it, the escrow releases the NFT to the buyer and the payment to the seller, atomically.

### What happens when I do this?
The marketplace becomes trustless — neither buyer nor seller can cheat. The blockchain enforces the trade.

### Prompt:

```
Implement these 3 marketplace methods in HeliusSolanaChainProvider:

1. createMarketplaceEscrow(listing: MarketplaceListing)
   - Lock the seller's NFT in a program-owned escrow
   - Option A (simpler): Use token delegation — the marketplace program 
     becomes the delegate for the NFT
   - Option B (proper): Transfer the NFT to a PDA (program-derived address)
   - Return the tx signature

2. cancelMarketplaceEscrow(listing: MarketplaceListing)
   - Return the NFT from escrow back to the seller
   - Return the tx signature

3. settleMarketplacePurchase(listing: MarketplaceListing, buyerWallet: string)
   - One atomic transaction that:
     a. Transfers payment (SOL/GRID) from buyer to seller
     b. Deducts the 5% marketplace fee
     c. Transfers the NFT from escrow to buyer
   - All or nothing — if any step fails, everything rolls back
   - Return the tx signature

For V1 (acceptable shortcut from docs/onchain.md):
- Use a server-escrowed flow: server holds the NFT in a treasury wallet
- Two separate transfers coordinated by the server
- Less trustless, but way simpler to build first
- Document the risk, upgrade to a proper program later
```

---

## Task 4.5 — Implement Trade Settlement

### What is this?
When two players complete a P2P trade, both sides' items need to swap atomically on-chain.

### Prompt:

```
Implement these 2 trade methods in HeliusSolanaChainProvider:

1. createTradeEscrow(session: TradeSession)
   - Lock both players' offered items in escrow
   - Return tx signature

2. settleTrade(session: TradeSession)
   - Atomic swap: both sides' items transfer simultaneously
   - If player A offered 5 Scrap + Drift Coupe, and player B offered 
     a Turbo Kit: everything swaps in one transaction
   - Return tx signature

For V1 (acceptable shortcut):
- Server-escrowed two-transfer flow (documented risk in docs/onchain.md)
- Server receives items from both sides, then distributes
- Upgrade to an atomic swap program later
```

---

## Task 4.6 — Implement Webhook Subscription

### What is this?
Helius can send your server a notification (webhook) when a transaction confirms on-chain. This is how the game knows "okay, that purchase actually went through on the blockchain, NOW update the database."

### What happens when I do this?
The game follows the golden rule: **confirm on chain, then commit in database**. No optimistic updates, no lost items.

### Prompt:

```
Implement subscribeToAssetEvents in HeliusSolanaChainProvider.

This is the webhook ingestion pipeline:

1. Create a Next.js API route (e.g., app/api/helius-webhook/route.ts)
   that receives POST requests from Helius

2. When Helius sends a webhook:
   a. Verify the webhook signature (Helius signs them)
   b. Parse the transaction details
   c. Call the registered callback with the asset info
   d. The game logic then updates the DB 
      (marks listing as "sold", transfers ownership, etc.)

3. Set up the webhook in Helius dashboard:
   - Go to your Helius dashboard → Webhooks
   - Create a webhook pointing to your API route URL
   - Subscribe to transaction events for your program/treasury address

4. subscribeToAssetEvents(cb) should:
   - Store the callback function
   - Return a cleanup function that removes it
   - The API route calls all stored callbacks when it receives an event
```

### Alternative options for chain confirmation:
| Option | Pros | Cons |
|--------|------|------|
| **Helius Webhooks (recommended)** | Real-time, reliable, Helius handles retry | Need a public URL for webhooks |
| **Polling getSignatureStatuses** | No webhook setup needed | Slower, wastes API calls, can miss things |
| **Helius Enhanced Websockets** | Real-time, no public URL needed | More complex, connection management |

---

## Task 4.7 — Test Phase 4

### How to do it:

```bash
# 1. Set env vars:
CHAIN_PROVIDER=helius
HELIUS_API_KEY=your-key
SOLANA_RPC_URL=https://devnet.helius-rpc.com/?api-key=your-key
SOLANA_NETWORK=devnet
TREASURY_WALLET=your-devnet-wallet

# 2. Get devnet SOL for testing:
# Visit https://faucet.solana.com

# 3. Start the game
pnpm dev

# 4. Test sequence:
#    a. Craft the Neon Wraith → should mint a real cNFT on devnet
#    b. Check your Phantom wallet (devnet) → you should see the NFT
#    c. List it on the marketplace → escrow should lock it
#    d. Buy it from another account → settlement should transfer it
#    e. Check Solana Explorer for your transactions
```

---

# Summary: The Complete Checklist

```
Phase 0: Web Restructuring & Hybrid Backend (1-2 days)
  [  ] 0.1  Frontend Routing & Pages (Homepage, Game, Auth, Whitepaper)
  [  ] 0.2  Hybrid Backend Storage Model (Memory + Supabase wrappers)

Phase 1: Accounts & Keys (30 min, no code)
  [  ] 1.1  Create Supabase project, get URL + keys
  [  ] 1.2  Run schema.sql in Supabase SQL editor
  [  ] 1.3  Create Upstash Redis, get URL + token
  [  ] 1.4  Get Helius API key ready (you have this)
  [  ] 1.5  Create .env file with all keys

Phase 2: Supabase Backend (2-3 days)
  [  ] 2.1  Install @supabase/supabase-js
  [  ] 2.2  Implement SupabaseAuthProvider
  [  ] 2.3  Implement SupabasePlayerRepository
  [  ] 2.4  Implement SupabaseInventoryRepository
  [  ] 2.5  Implement SupabaseMarketplaceRepository
  [  ] 2.6  Implement SupabaseTradeRepository
  [  ] 2.7  Implement SupabaseEconomyLedgerRepository
  [  ] 2.8  Wire repositories through the registry
  [  ] 2.9  Test Phase 2

Phase 3: Upstash Redis (2-4 hours)
  [  ] 3.1  Install @upstash/redis
  [  ] 3.2  Implement UpstashRedisCacheProvider (4 methods, ~20 lines)
  [  ] 3.3  Test Phase 3

Phase 4: Helius Solana (3-5 days)
  [  ] 4.1  Install Solana dependencies
  [  ] 4.2  Implement wallet + balance methods
  [  ] 4.3  Implement minting (compressed NFTs)
  [  ] 4.4  Implement marketplace escrow
  [  ] 4.5  Implement trade settlement
  [  ] 4.6  Implement webhook subscription
  [  ] 4.7  Test Phase 4
```

---

# Quick Reference: All Environment Variables

```bash
# === ADAPTER SELECTION (which backend to use) ===
DB_PROVIDER=supabase          # memory | supabase
CACHE_PROVIDER=redis          # noop | redis
CHAIN_PROVIDER=helius         # mock | helius
AUTH_PROVIDER=supabase        # guest | supabase

# === SUPABASE ===
SUPABASE_URL=https://xxxxx.supabase.co
SUPABASE_ANON_KEY=eyJhbG...
SUPABASE_SERVICE_ROLE_KEY=eyJhbG...

# === UPSTASH REDIS ===
UPSTASH_REDIS_REST_URL=https://xxxxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=AXxx...

# === HELIUS / SOLANA ===
HELIUS_API_KEY=your-key
SOLANA_RPC_URL=https://devnet.helius-rpc.com/?api-key=your-key
SOLANA_NETWORK=devnet         # devnet | mainnet-beta
TREASURY_WALLET=your-wallet-address
MARKETPLACE_PROGRAM_ID=       # optional until escrow program ships

# === REALTIME ===
NEXT_PUBLIC_REALTIME_PROVIDER=ws    # preview | ws | colyseus
NEXT_PUBLIC_WS_URL=ws://localhost:2567

# === PRODUCTION SAFETY ===
NEXT_PUBLIC_DEV_FAST_GRIND=false    # MUST be false in production
NEXT_PUBLIC_ENABLE_ADMIN=false       # MUST be false in production
```

---

# One Last Thing: What Can You Ship Without?

| Without This | Game Still Works? | What's Missing |
|-------------|-------------------|----------------|
| Supabase | ⚠️ Yes, but data resets every restart | No persistence at all |
| Redis | ✅ Yes | Rate limits use in-memory Map (good enough for 1 server) |
| Helius | ✅ Yes | Everything uses mock chain (fake transactions) |
| Colyseus | ✅ Yes | Use WebSocket server (1 room, 1 server, up to ~100 players) |

**Minimum viable production:** Supabase only. Redis is nice-to-have. Helius is only needed when you want real blockchain. Colyseus is only needed when you want 1000+ players.

---

> [!TIP]
> **You can ask me to implement any of these tasks for you.** Just say something like "Do Task 2.3" or "Implement the Supabase player repository" and I'll write the actual code.
