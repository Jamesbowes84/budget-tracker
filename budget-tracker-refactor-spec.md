# Budget Tracker v2 — Architecture Refactor Spec

**File:** `index.html` (single file, ~7,000 lines, vanilla JS, localStorage + Dropbox sync)
**Hosted:** GitHub Pages at `jamesbowes84.github.io/budget-tracker/`
**Goal:** Single source of truth for categories, groups, and budgets. Recurring rules ARE the budget. Math adds up at every time horizon.

---

## 1. WHAT'S WRONG TODAY

### Data Duplication
| Concept | Currently lives in | Problem |
|---------|-------------------|---------|
| Categories | Hardcoded in `autoCategory()`, scattered in `state.needs[].name`, `state.wants[].name`, keyword rules in `bowes_cat_overrides` | Change one, others stale |
| Budget amounts | `state.needs[].amt`, `state.wants[].amt` | Separate from recurring — must sync manually |
| Recurring amounts | `state.recurring[].amount` | Separate from budget — editing one doesn't update the other |
| Need/Want classification | Implicit — if it's in `state.needs` it's a need | Can't change without moving data between arrays |
| Groups (Housing, Transport) | `state.needs[].group` only — wants have no groups | Inconsistent |
| Debt balances | `state.debts[].balance` | Must manually enter same balance on Net Worth page |
| CC balance | `state.ccBalance` AND `state.debts[].balance` | Two sources for same number |

### Missing Connections
- No way to see: "I make $6,248/mo, my recurring obligations are $5,400, I have $848 free" at a glance
- Budget shows monthly averages but actual payments are lumpy (semi-annual insurance, biweekly pay)
- Can't answer: "What's my budget look like THIS specific month with THIS specific payday schedule?"
- Debts don't flow into net worth
- No single category management page

---

## 2. TARGET DATA MODEL

### `state.categories` — Master category list
```js
state.categories = [
  { id: "cat_001", name: "Electric", groupId: "grp_utilities" },
  { id: "cat_002", name: "Phone", groupId: "grp_utilities" },
  { id: "cat_003", name: "Fuel", groupId: "grp_transport" },
  { id: "cat_004", name: "Dining Out", groupId: "grp_food" },
  { id: "cat_005", name: "Mortgage", groupId: "grp_housing" },
  // ... every spend category lives here
  // Special categories:
  { id: "cat_income", name: "Income", groupId: null },
  { id: "cat_transfer", name: "Transfer", groupId: null },
  { id: "cat_uncat", name: "Uncategorized", groupId: null },
];
```

### `state.groups` — Budget groups with need/want tag
```js
state.groups = [
  { id: "grp_housing",    name: "Housing",     type: "need" },
  { id: "grp_utilities",  name: "Utilities",   type: "need" },
  { id: "grp_transport",  name: "Transport",   type: "need" },
  { id: "grp_food",       name: "Food & Home", type: "need" },
  { id: "grp_kids",       name: "Kids",        type: "need" },
  { id: "grp_taxes",      name: "Taxes",       type: "need" },
  { id: "grp_recreation", name: "Recreation",  type: "want" },
  { id: "grp_personal",   name: "Personal",    type: "want" },
  // ...
];
```

### `state.recurring` — THE source of truth for planned spend (unchanged structure, but now IS the budget)
```js
state.recurring = [
  {
    name: "Electric",
    billType: "variable",    // fixed | variable | income
    amount: 180,
    variance: 30,            // ± for variable bills
    category: "cat_001",     // REFERENCES state.categories.id
    schedule: "monthly",     // monthly | biweekly | weekly | bimonthly | semiannual | annual
    day: 15,                 // schedule-specific
    start: "2025-01-01",
    end: "",
    notes: "Northwestern REC",
    active: true
  },
  // Income rules too:
  {
    name: "Mom Brands Paycheck",
    billType: "income",
    amount: 2885,
    category: "cat_income",
    schedule: "biweekly",
    biweeklyStart: "2025-01-03",
    biweeklyDow: 5,
    active: true
  }
];
```

### `state.debts` — Feeds net worth automatically
```js
state.debts = [
  {
    id: "debt_001",
    name: "PNC Credit Card",
    type: "credit_card",
    balance: 7398,
    original: 35000,
    rate: 9.9,
    minPayment: 500,
    payment: 500,           // planned monthly
    notes: "PNC",
    // NEW: links to net worth
    nwLiability: true       // auto-appears as liability on Net Worth
  }
];
```

### `state.dayItems` — One-off planned events (unchanged)
```js
state.dayItems = [
  { date: "2026-07-15", desc: "Vacation", amount: 2000, type: "expense" }
];
```

### What gets REMOVED from state
- `state.needs` → replaced by: `state.recurring` filtered by group type === "need"
- `state.wants` → replaced by: `state.recurring` filtered by group type === "want"
- `state.subscriptions` → already merged into recurring
- `state.roadmap` → replaced by Avalanche Planner (already done)
- `state.ccBalance`, `state.ccOriginal` → use `state.debts` CC entry
- `state.income` → computed from income recurring rules

### Computed values (functions, not stored)
```js
// Monthly income = sum of all income recurring rules, normalized to monthly
function monthlyIncome() { ... }

// Budget for a category in a specific month
function budgetForMonth(categoryId, year, month) {
  // Find recurring rules with this category
  // Check if they fire in this month
  // Return the actual amount due (not monthly average)
}

// All needs = recurring rules whose category's group is type "need"
function allNeeds() {
  return state.recurring.filter(r => {
    var cat = catById(r.category);
    var grp = cat ? groupById(cat.groupId) : null;
    return grp && grp.type === 'need' && r.active && r.billType !== 'income';
  });
}

// Free cash = income - all recurring expenses (monthly normalized)
function freeCash() { ... }
```

---

## 3. MIGRATION STRATEGY

User has existing data in `bowes_budget_v1` localStorage key. Must migrate on first load of v2.

```js
function migrateToV2(oldState) {
  // 1. Build categories from existing needs, wants, and autoCategory keywords
  //    De-duplicate by name
  
  // 2. Build groups from existing needs[].group + infer for wants
  //    Tag each group as need or want based on which array it came from
  
  // 3. Convert needs/wants budget items to recurring rules
  //    - Match by name to existing recurring rules
  //    - If recurring rule exists: keep rule, ensure category link
  //    - If no rule exists: create a monthly rule from the budget item
  
  // 4. Link debts to net worth (add nwLiability flag)
  
  // 5. Convert category overrides from bowes_cat_overrides
  //    Map old category names → new category IDs
  
  // 6. Re-tag existing transactions with category IDs
  //    Keep old .category string as fallback
  
  // 7. Set state.schemaVersion = 2
  
  return newState;
}
```

**Key principle:** Never lose data. Old fields stay as fallbacks. New fields get added. `schemaVersion` flag prevents re-migration.

---

## 4. PAGE-BY-PAGE CHANGES

### NEW: Categories page (under Budget nav section)
- Master list of all spend categories
- Each shows: name, group assignment (dropdown), keyword rules for auto-import
- Add/edit/delete categories
- Merge two categories (reassigns all transactions)

### NEW: Groups page (or section within Categories)
- List of groups: Housing, Utilities, Transport, etc.
- Each tagged Need or Want (toggle)
- Drag to reorder display priority
- Add/rename/delete groups

### CHANGED: Needs page → "Needs" view
- Now a READ-ONLY VIEW of recurring rules filtered by group.type === "need"
- Grouped by group
- Each row shows: category name, recurring rule amount, schedule, monthly equivalent
- Click any row → opens the recurring rule editor (not a separate budget editor)
- Group totals, grand total
- "Add" button → opens recurring rule modal with group pre-filtered

### CHANGED: Wants page → "Wants" view  
- Same as Needs but filtered by group.type === "want"

### CHANGED: Dashboard
- Income card: computed from income recurring rules for this specific month
- Spent card: actual transactions this month
- Budget card: sum of all recurring rules that fire THIS month (period-aware)
- Free Cash card: income minus all recurring (monthly normalized)
- **NEW: Surplus/Deficit indicator** — "You're $200 over-committed" or "$848 unallocated"
- Progress bars: compare actual spend per category against the rule amount for THIS month

### CHANGED: Monthly P&L
- Budget column: actual rule amounts for the selected month (not monthly averages)
- Shows which categories have a rule firing this month and which don't
- Bottom line: income rules this month minus expense rules this month = planned surplus/deficit
- Actual column: real transactions
- Variance: planned vs actual

### CHANGED: Recurring page
- Now THE budget management page
- Shows all rules grouped by Need/Want/Income
- Budget Sync button removed (no longer needed — recurring IS the budget)
- Summary cards: Total Income, Total Needs, Total Wants, Free Cash, Surplus/Deficit
- Each rule row shows: name, category, schedule, amount, monthly equiv, next due date

### CHANGED: Budget Plan page
- Becomes the "big picture" view
- Shows: annual income, annual needs, annual wants, annual free cash
- Monthly, weekly, daily breakdowns
- Calendar heat map: which days of the month have the most bills
- "What if" slider: "If I cut $X from wants, I'm debt-free Y months sooner"

### CHANGED: Day by Day
- Checking balance projection unchanged
- Now also shows: "This month's budget status" mini-summary
- Each day shows which recurring rules fire (with exact amounts, not estimates)

### CHANGED: Debts page
- Same UI
- Debts with `nwLiability: true` auto-appear on Net Worth as liabilities
- When debt balance changes (import, manual edit), net worth updates automatically

### CHANGED: Net Worth page
- Liabilities section auto-populated from debts
- Manual liabilities still allowed for things not in debts
- Assets unchanged (manual entry)

### CHANGED: Avalanche Planner
- Free cash auto-populates from the new computed function
- No manual entry needed

### CHANGED: Alerts
- Over-budget alerts use period-aware budgetForMonth()
- NEW: "Over-committed" alert when total recurring > income
- NEW: "Unbudgeted spending" alert when a category has transactions but no recurring rule

### CHANGED: Uncategorized / Import
- Category dropdowns pull from `state.categories`
- Keyword rules stored on the category object (not separate localStorage)
- Import auto-categorizer uses category keyword rules instead of hardcoded function

---

## 5. WHAT STAYS THE SAME

- Single HTML file
- localStorage with `bowes_budget_v1` key (just new schema inside)
- Dropbox sync (same mechanism)
- Chart.js for charts
- All CSS / dark theme
- Transaction structure (date, description, category, amount, type, account, pending)
- Day by Day checking balance projection
- Avalanche simulation engine
- Import parsers (PNC + CC)
- Mobile bottom nav

---

## 6. EXECUTION PHASES

### Phase 1: Data Model + Migration (Foundation)
- Add `state.categories`, `state.groups`, `state.schemaVersion`
- Write `migrateToV2()` function
- Update `loadState()` to call migration on first load
- Update `allCategories()` to read from `state.categories`
- Update `resolveCategory()` to use category keyword rules
- All existing features keep working with new data model

### Phase 2: Categories & Groups UI
- Build Categories management page
- Build Groups management page (or section)
- Update nav
- Category dropdowns everywhere pull from master list

### Phase 3: Recurring = Budget
- Remove `state.needs` and `state.wants` as separate data stores
- Needs page becomes a view of recurring rules (filtered by group type)
- Wants page becomes a view of recurring rules (filtered by group type)
- Remove Budget Sync entirely (no longer needed)
- Update `budgetForMonth()` to work from recurring rules directly

### Phase 4: Budget Reality Views
- Dashboard: period-aware budget cards, surplus/deficit
- Monthly P&L: actual rule amounts per month
- Budget Plan: annual/monthly/weekly/daily roll-up
- Day by Day: which rules fire each day with amounts

### Phase 5: Debt ↔ Net Worth Link
- Debts auto-feed net worth liabilities
- Balance changes propagate automatically

---

## 7. RISKS & MITIGATIONS

| Risk | Mitigation |
|------|-----------|
| Migration corrupts existing data | `migrateToV2()` deep-copies first, sets `schemaVersion`, never deletes old fields until v3 |
| Dropbox sync conflict between v1 and v2 | Check `schemaVersion` on Dropbox load — if mismatch, migrate the older one |
| Recurring rules don't cover all budget items | Migration creates placeholder rules for any budget items without a matching recurring rule |
| Category ID references break | Keep `.category` as string name as fallback; lookup by name if ID not found |
| User has no recurring rules set up yet | Needs/Wants pages show "Set up recurring rules to populate your budget" with a link |

---

## 8. CURRENT FILE MAP (for Claude Code context)

```
Lines 1–540:      CSS (all styles, dark theme, responsive)
Lines 540–610:    Sidebar HTML (nav structure)
Lines 610–1100:   Page HTML (dashboard through misc)
Lines 1100–1200:  Recurring page HTML
Lines 1200–1700:  Modal HTML (budget, tx, recurring, debt, etc.)
Lines 1700–1840:  DEFAULT_STATE + loadState/saveState
Lines 1840–1900:  Page routing (nav function, renderPage map)
Lines 1900–2000:  Utility functions (fmt, dates, categories)
Lines 2000–2200:  Dashboard render + helpers
Lines 2200–2500:  Grouped progress bars + dashboard cards
Lines 2500–2600:  Weekly view
Lines 2600–2700:  Monthly P&L
Lines 2700–2800:  Budget Plan render
Lines 2800–2900:  Budget save/edit/frequency helpers
Lines 2900–3100:  Avalanche simulation engine
Lines 3100–3400:  Import parsers (PNC, CC, CSV)
Lines 3400–3600:  Transaction modal + import preview
Lines 3600–3800:  Confirm import + dupe detection
Lines 3800–4100:  Uncategorized page + category rules
Lines 4100–4200:  autoCategory() hardcoded keywords
Lines 4200–4400:  Recurring rules engine (schedule matching)
Lines 4400–4600:  Recurring CRUD functions
Lines 4600–4800:  Budget sync functions
Lines 4800–5000:  Day by Day balance engine
Lines 5000–5200:  Net Worth + Goals
Lines 5200–5400:  Debts page render
Lines 5400–5600:  Alerts engine
Lines 5600–5800:  Reports (charts, trends)
Lines 5800–6000:  Bill Calendar
Lines 6000–6200:  Dropbox sync
Lines 6200–6400:  Mobile nav + init
Lines 6400–6600:  Event listeners + DOMContentLoaded
```
