# 🌙 Evening AI

> *Tell it what you're planning. It handles everything else.*

**Evening AI** is the first agentic evening planner built on Swiggy's MCP platform — the only project that chains all three Swiggy MCP servers (Food, Instamart, and Dineout) in a single conversational flow.

One prompt. Three purchase flows. Zero friction.

```
"I have a date tonight at 8pm. Italian food, budget ₹2500, she likes candles."

→ Books a table at Trattoria on Indiranagar 100ft Road via Dineout     ✓
→ Queues Instamart: red wine, scented candles, flowers (arrives 7pm)   ✓
→ Schedules Swiggy dessert order to arrive at 10:30pm                  ✓
```

---

## What is this?

Planning a good evening involves at least three different apps, multiple searches, and a bunch of decisions under time pressure. Evening AI collapses that into a single conversation.

It's built on [Swiggy Builders Club](https://mcp.swiggy.com/builders/) — a developer program that exposes Swiggy's Food, Instamart, and Dineout platforms as MCP (Model Context Protocol) servers. Evening AI orchestrates all three in one agentic loop, using Claude as the reasoning engine.

---

## Architecture

```
User Prompt
     │
     ▼
┌─────────────────────────────────────┐
│         Evening AI Agent            │
│         (Claude Sonnet 4)           │
│                                     │
│  1. Parse intent & constraints      │
│  2. Plan across all three services  │
│  3. Execute in parallel             │
│  4. Confirm & summarise             │
└──────┬──────────────┬───────────────┘
       │              │              │
       ▼              ▼              ▼
  Dineout MCP    Instamart MCP   Food MCP
  ───────────    ─────────────   ────────
  search         search_         search_
  _restaurants   products        restaurants
  _dineout                       _menu
       │              │              │
  get_available   update_cart    update_food
  _slots                         _cart
       │              │              │
  book_table      checkout       place_food
                                  _order
                                 (scheduled)
```

The agent runs three MCP server connections concurrently. The Dineout booking and the Instamart delivery are dispatched in parallel; the Food dessert order is placed with a scheduled delivery window that the agent calculates from the booking time.

---

## Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Agent brain | Claude `claude-sonnet-4-20250514` via Anthropic API | Best-in-class tool use and multi-step reasoning |
| MCP client | `@anthropic-ai/sdk` (MCP client mode) | Native support for chaining multiple MCP servers |
| MCP servers | `mcp.swiggy.com/food`, `/im`, `/dineout` | Swiggy Builders Club production APIs |
| Backend | Node.js + Express | Lightweight, great async/streaming support |
| Frontend | React + Tailwind | Clean conversational UI |
| Auth | OAuth2 with PKCE | Per Swiggy's MCP auth spec |
| Scheduling | node-cron + Swiggy order window API | Timed dessert delivery |

---

## Agent Design

Evening AI uses a **plan-then-execute** pattern with three phases:

### Phase 1 — Intent parsing
The agent extracts structured intent from the user's natural language prompt:
- Occasion type (date, friends over, solo, celebration)
- Time and date
- Budget (total, or split across services)
- Cuisine preferences and dietary restrictions
- Vibe / ambiance preferences

### Phase 2 — Parallel planning
Three sub-tasks are planned simultaneously:
1. **Dineout**: `search_restaurants_dineout` → `get_available_slots` → score options by budget, cuisine, rating, and proximity
2. **Instamart**: `search_products` across ambiance items (drinks, candles, flowers, snacks) — mapped to occasion type
3. **Food**: `search_restaurants` + `search_menu` for a late-night dessert option matching cuisine and dietary preferences

### Phase 3 — Execution with confirmation
The agent presents a complete plan before committing any transactions:

```
Here's what I've planned for tonight:

🍽  Dineout — Prego, UB City (8:00 PM, table for 2)             ₹200 cover
🛒  Instamart — 1x Sula Brut, 3x soy candles, 12 roses         ₹890 (arrives 7pm)
🍮  Swiggy — Tiramisu × 2 from Smoke House Deli (arrives 10:30pm)  ₹480

Total: ₹1,570 of your ₹2,500 budget

Confirm all three?  [Yes, book everything]  [Adjust]
```

One confirmation triggers all three purchase flows.

---

## Project Structure

```
evening-ai/
├── agent/
│   ├── index.ts              # Main agent loop
│   ├── intent-parser.ts      # Extracts structured intent from NL
│   ├── planner.ts            # Parallel planning across 3 services
│   ├── executor.ts           # MCP tool dispatch & confirmation
│   └── scheduler.ts          # Calculates delivery windows
├── mcp/
│   ├── food-client.ts        # Swiggy Food MCP connection
│   ├── instamart-client.ts   # Swiggy Instamart MCP connection
│   └── dineout-client.ts     # Swiggy Dineout MCP connection
├── server/
│   ├── app.ts                # Express server
│   ├── auth.ts               # OAuth2 + PKCE flow
│   └── routes.ts             # API endpoints
├── frontend/
│   ├── src/
│   │   ├── App.tsx
│   │   ├── Chat.tsx          # Conversational UI
│   │   └── PlanCard.tsx      # 3-service plan preview
│   └── ...
├── .env.example
└── README.md
```

---

## Getting Started

> **Note:** This project requires Swiggy Builders Club API access. [Apply here](https://mcp.swiggy.com/builders/access/).

### Prerequisites
- Node.js 20+
- Swiggy Builders Club API credentials (Food + Instamart + Dineout)
- Anthropic API key

### Install

```bash
git clone https://github.com/YOUR_USERNAME/evening-ai
cd evening-ai
npm install
cp .env.example .env
```

### Configure `.env`

```env
ANTHROPIC_API_KEY=your_key_here
SWIGGY_CLIENT_ID=your_swiggy_client_id
SWIGGY_CLIENT_SECRET=your_swiggy_client_secret
SWIGGY_REDIRECT_URI=http://localhost:3000/callback
PORT=3000
```

### Run

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) and try:

```
"I have friends coming over at 7. North Indian food, budget ₹3000, need drinks too"
```

---

## Example Flows

### Date night
```
Input:  "Date tonight at 8. She likes Thai. Budget ₹2000. Make it nice."

Output:
  → Dineout: Nara Thai, Indiranagar (8:00 PM, table for 2)
  → Instamart: prosecco × 1, pillar candles × 3, lily bouquet × 1
  → Food: Mango sticky rice × 2 from Nara Thai (scheduled 10:45 PM)
```

### Friends over
```
Input:  "5 friends coming at 7. Pizzas and starters. Need beer and mixers too."

Output:
  → Food: Margherita × 2, Pepperoni × 2, Loaded fries × 2 (arrives 6:50 PM)
  → Instamart: Kingfisher × 6, Sprite × 2, soda × 4, chips × 3
  → Dineout: (skipped — staying in)
```

### Solo night in
```
Input:  "Quiet night in. Something comforting. Rainy evening."

Output:
  → Food: Dal makhani + butter naan + kheer from Punjabi by Nature
  → Instamart: masala chai sachets × 5, dark chocolate × 2
  → Dineout: (skipped — staying in)
```

---

## Why this exists

Evening planning is a genuinely annoying multi-app problem. Before Evening AI:

1. Open Dineout — search, filter, check availability, book
2. Open Instamart — search for ambiance items separately
3. Open Swiggy — find a dessert place, set a reminder to order later

After Evening AI: one sentence.

The deeper point: this project exists to demonstrate what becomes possible when Swiggy's three platforms are treated as a unified API surface rather than three separate apps. Builders Club makes this possible for the first time.

---

## Roadmap

- [ ] Core agent + 3-server chaining (in progress)
- [ ] Web UI with plan preview card
- [ ] Calendar integration — auto-detect "date night" from calendar events
- [ ] Budget memory — learns your typical spend across occasion types
- [ ] WhatsApp interface — send one message, get the whole evening planned
- [ ] Occasion templates — "anniversary", "birthday dinner", "client entertainment"
- [ ] Post-evening summary — "Here's what you spent, here's a photo memory prompt"

---

## Built on Swiggy Builders Club

This project is being built as part of [Swiggy Builders Club](https://mcp.swiggy.com/builders/) — Swiggy's developer program giving access to their Food, Instamart, and Dineout MCP APIs. Evening AI is designed to showcase the full depth of the platform: all three servers, chained in a single agentic flow.

---

## Author

Built by [Rishabh Jain](https://github.com/rishj606) — [LinkedIn](https://www.linkedin.com/in/rishabh-jain-a95997173/)

Reach out: builders@swiggy.in (for Builders Club collaboration)

---

*Evening AI is an independent project and is not affiliated with or endorsed by Swiggy. Built using the Swiggy Builders Club developer program.*
