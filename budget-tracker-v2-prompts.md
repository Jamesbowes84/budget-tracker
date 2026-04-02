# Budget Tracker v2 — Claude Code Execution Prompts

Use these prompts sequentially in Claude Code. Each phase builds on the previous. Test after each phase before moving to the next.

**Setup:** Open the project directory containing `index.html` in Claude Code.

---

## PRE-FLIGHT: Read the spec

```
Read the file budget-tracker-refactor-spec.md in this directory. This is the architecture spec for a data model refactor of index.html. Confirm you understand the current data model, the target data model, and the migration strategy before we begin. Don't make any changes yet.
```

---

## PHASE 1: Data Model + Migration

```
We're refactoring the budget tracker (index.html) per the spec in budget-tracker-refactor-spec.md. This is Phase 1: Data Model + Migration.

CONTEXT: This is a single-file vanilla JS app (~7,000 lines). All state lives in localStorage key 'bowes_budget_v1'. There's also 'bowes_cat_overrides' and 'bowes_amt_rules' in separate localStorage keys. Dropbox sync pushes/pulls the same state object.

DO THE FOLLOWING:

1. Add new fields to DEFAULT_STATE:
   - state.categories = [] (array of {id, name, groupId, keywords[]})
   - state.groups = [] (array of {id, name, type: 'need'|'want'})
   - state.schemaVersion = 2

2. Write a migrateToV2(oldState) function that:
   a. Skips if state.schemaVersion >= 2
   b. Extracts unique category names from: state.needs[].name, state.wants[].name, all transaction .category values, and the autoCategory() keyword mappings
   c. Builds state.categories with generated IDs (use 'cat_' + lowercase name with spaces replaced by underscores)
   d. For each category, pulls keyword rules from bowes_cat_overrides AND from the autoCategory() function's hardcoded mappings. Store as keywords[] array on the category object. Example: {id:'cat_fuel', name:'Fuel', groupId:'grp_transport', keywords:['KWIK FILL','SHEETZ','SUNOCO',...]}
   e. Builds state.groups from existing needs[].group values (each tagged type:'need') plus infers groups for wants (default group 'Wants' tagged type:'want', or specific groups if identifiable)
   f. For each existing needs[] and wants[] budget item, finds or creates a matching recurring rule:
      - If state.recurring already has a rule with matching name → link it to the category, keep the rule
      - If no matching rule exists → create a monthly recurring rule from the budget item (amount=item.amt, schedule='monthly', day=1, category=matched category ID)
      - Preserve freq/rawAmt if they exist on the budget item
   g. Marks all debts with nwLiability:true
   h. Sets state.schemaVersion = 2
   i. Does NOT delete state.needs or state.wants yet (kept as fallback for Phase 3)

3. Update loadState() to call migrateToV2(state) after loading from localStorage

4. Update allCategories() to read from state.categories instead of state.needs/wants names. Return sorted name strings (same interface as before so nothing breaks).

5. Update resolveCategory(desc, amount) to:
   - First check bowes_amt_rules (unchanged)
   - Then check bowes_cat_overrides (unchanged) 
   - Then check state.categories[].keywords for matches (replaces autoCategory())
   - Keep autoCategory() as final fallback for safety
   - Return the category name string (same interface)

6. Ensure loadState() also initializes state.categories and state.groups if missing (backwards compat)

TEST: After these changes, the app should load normally. Existing data should auto-migrate. All pages should render without errors. Open the browser console and verify no errors. Verify state.schemaVersion === 2 in localStorage.

IMPORTANT: Do NOT change any page rendering or UI yet. This phase is data model only. Every existing page should work exactly as before.
```

---

## PHASE 2: Categories & Groups UI

```
Continuing the refactor per budget-tracker-refactor-spec.md. This is Phase 2: Categories & Groups Management UI.

Phase 1 is complete — state.categories and state.groups exist and are populated.

DO THE FOLLOWING:

1. Add a "Categories" page (id="page-categories"):
   HTML: A page with two sections:
   
   Section 1 — Groups:
   - Table showing all groups: Name, Type (need/want toggle button), Category count, Edit/Delete buttons
   - "+ Add Group" button
   - Each group's type toggles between need/want with a single click (green badge for need, amber for want)
   
   Section 2 — Categories:
   - Table showing all categories: Name, Group (dropdown of all groups), Keywords (comma-separated, editable), Transaction count, Edit/Delete
   - "+ Add Category" button
   - Inline editing: clicking the group dropdown or keyword field changes it immediately
   - "Merge" action: select two categories → merge all transactions from one into the other, delete the source

2. Add nav item for Categories:
   - In the Budget nav section: add "Categories" between current items
   - Icon: ◧

3. Add page routing:
   - 'categories': renderCategories
   - Write renderCategories() function

4. Add modal for category edit:
   - Fields: Name, Group (dropdown), Keywords (textarea, one per line)
   - Save updates state.categories, re-renders

5. Add modal for group edit:
   - Fields: Name, Type (need/want radio)
   - Save updates state.groups, re-renders

6. Update category dropdowns app-wide:
   - catOptions() already reads from allCategories() which reads state.categories — verify this works
   - The recurring rule modal's category dropdown should also include a "+ New Category" option that creates one on the fly

TEST: Navigate to Categories page. Add a group. Add a category. Change a category's group. Edit keywords. Verify the category appears in import dropdowns and transaction edit modal.
```

---

## PHASE 3: Recurring = Budget (the big one)

```
Continuing the refactor per budget-tracker-refactor-spec.md. This is Phase 3: Recurring Rules become the single source of truth for the budget. This is the most impactful phase.

Phases 1-2 are complete — categories, groups, and migration are in place.

DO THE FOLLOWING:

1. Add helper functions that derive budget data from recurring rules:

   function rulesForGroup(groupId) — returns active non-income recurring rules whose category belongs to this group
   
   function needRules() — returns all active non-income recurring rules whose category's group has type 'need'
   
   function wantRules() — returns all active non-income recurring rules whose category's group has type 'want'
   
   function incomeRules() — returns all active recurring rules where billType === 'income'
   
   function monthlyIncome() — sum of all income rules normalized to monthly
   
   function monthlyNeeds() — sum of all need rules normalized to monthly
   
   function monthlyWants() — sum of all want rules normalized to monthly
   
   function freeCash() — monthlyIncome() - monthlyNeeds() - monthlyWants()
   
   function budgetForCategoryInMonth(categoryName, y, m) — finds recurring rules for this category, checks if they fire in month y/m, returns the actual amount due that month (full amount for non-monthly items in payment months, $0 in non-payment months). This REPLACES the old budgetForMonth(item, y, m).
   
   function totalBudgetForMonth(y, m) — sum of budgetForCategoryInMonth for all categories in that month

2. Rewrite renderNeeds():
   - Get all need groups: state.groups.filter(g => g.type === 'need')
   - For each group, get its recurring rules via rulesForGroup(g.id)
   - Render the same grouped/collapsible table but sourced from recurring rules
   - Each row shows: rule name, amount (with frequency note for non-monthly), schedule, next due date
   - Click Edit → opens the recurring rule editor (openEditRecurring), NOT the old budget item modal
   - "+ Add" button → openAddRecurring() with category pre-set
   - Group totals show monthly normalized amounts
   - Grand total + monthly income + surplus/deficit at bottom
   
3. Rewrite renderWants():
   - Same approach as renderNeeds() but filtered by group.type === 'want'

4. Update renderGroupedProgress() (dashboard progress bars):
   - Instead of iterating state.needs/state.wants, iterate ALL categories that have recurring rules
   - Group them by their group
   - For each category: budget = budgetForCategoryInMonth(cat.name, y, m), spent = spentInMonth(cat.name, y, m)
   - Same dual-color bar logic

5. Update renderMonthly() (Monthly P&L):
   - Budget column: budgetForCategoryInMonth() for each category
   - Group by needs/wants using group.type
   - Total budget for month: totalBudgetForMonth(y, m)

6. Update renderDashboard():
   - Income card: actual income this month or monthlyIncome() if no transactions yet
   - Total budget: totalBudgetForMonth(y, m) 
   - Free Cash card: freeCash()
   - Add surplus/deficit note: monthlyIncome() - monthlyNeeds() - monthlyWants()

7. Update renderAlerts():
   - Over-budget check: use budgetForCategoryInMonth()
   - Add new alert: if monthlyNeeds() + monthlyWants() > monthlyIncome() → urgent "Over-committed by $X/mo"

8. Update alert badge count to match

9. Remove old budget item modal (modal-budget-item) — no longer needed since editing happens through the recurring rule modal

10. Remove old budget save/edit functions: saveBudgetItem(), openEditBudget(), openAddNeed(), openAddWant(), deleteBudgetItem()
    Replace openAddNeed() with: openAddRecurring('fixed') with a need-group preset
    Replace openAddWant() with: openAddRecurring('fixed') with a want-group preset

11. Remove syncRecurringToBudget(), openBudgetSync(), applySyncMatches(), and all sync-related code — no longer needed since recurring IS the budget

12. Clean up state.needs and state.wants references:
    - Keep them in DEFAULT_STATE as empty arrays for backwards compat
    - loadState() no longer needs to initialize them
    - No rendering code should read from them

TEST: 
- Needs page shows all recurring rules grouped by need groups. Editing a row opens recurring editor.
- Wants page shows all recurring rules grouped by want groups.
- Dashboard progress bars match recurring rule amounts.
- Monthly P&L shows correct budget amounts per category per month.
- Budget Sync button is gone from Recurring page (not needed).
- Adding a recurring rule with a need-group category auto-appears on Needs page.
- Free Cash card on dashboard matches: income - needs - wants.
- Semi-annual items show $0 budget in non-payment months, full amount in payment months.
```

---

## PHASE 4: Budget Reality Views

```
Continuing the refactor per budget-tracker-refactor-spec.md. This is Phase 4: Budget Reality Views — making the numbers real at every time horizon.

Phases 1-3 are complete — recurring rules are the budget, categories/groups are managed.

DO THE FOLLOWING:

1. Rewrite renderBudgetPlan() as the "big picture" view:
   
   Summary cards (top row):
   - Annual Income: monthlyIncome() * 12
   - Annual Needs: monthlyNeeds() * 12
   - Annual Wants: monthlyWants() * 12  
   - Annual Free Cash: freeCash() * 12
   
   Breakdown table with multiple horizons:
   | Category | Daily | Weekly | Monthly | Annual |
   Daily = monthly / 30, Weekly = monthly / 4.33, Annual = monthly * 12
   Grouped by need/want groups
   Total row for each
   Grand total row showing surplus/deficit at each horizon
   
   Over-commitment warning:
   If monthlyNeeds() + monthlyWants() > monthlyIncome():
   Show a red banner: "Over-committed by $X/mo ($Y/yr). You need to cut $X from wants or increase income."

2. Update renderDayByDay() to show budget context:
   - Below the balance projection, add a mini-card row:
     "This month: $X budgeted | $Y spent so far | $Z remaining | Free cash: $W"
   - In the daily detail, show which recurring rules fire that day with exact amounts
     (already partially done — enhance with category and "budget item" labels)

3. Add a "This Month" summary section to the Dashboard (between cards and progress bars):
   - Compact row showing: "Apr 2026: 3 paydays ($8,655) | $4,200 in bills | $2,100 wants | $2,355 free"
   - This uses ACTUAL rule firing dates for the current month, not monthly averages
   - Flags months with unusual cash flow (3 paydays, semi-annual insurance due, etc.)

4. Update renderWeekly():
   - Weekly budget per category: use category's monthly amount / 4.33
   - Show "This week's bills" section: which recurring rules fire between Monday-Sunday
   - Running total: how much of the weekly budget is committed to specific bills this week

5. Update renderCalendar():
   - Each calendar day shows the recurring rule amounts scheduled for that day
   - Monthly total at top: sum of all rules firing this month
   - Color-code heavy bill days (multiple rules same day)

TEST:
- Budget Plan page shows annual/monthly/weekly/daily breakdown with correct math
- Day by Day shows budget context
- Dashboard shows month-specific summary (not averages)
- Weekly view shows this week's bills
- Calendar shows correct amounts per day
- Over-commitment warning appears if expenses > income
```

---

## PHASE 5: Debt ↔ Net Worth Link

```
Continuing the refactor per budget-tracker-refactor-spec.md. This is Phase 5: Connecting debts to net worth.

Phases 1-4 are complete.

DO THE FOLLOWING:

1. Update renderNetWorth():
   - Liabilities section: auto-populate from state.debts where nwLiability === true
   - Show each debt: name, balance, rate
   - Manual "Other liabilities" section below for anything not in debts
   - Total liabilities = sum of debt balances + manual liabilities
   - Keep manual assets section unchanged
   - Net worth = assets - liabilities

2. When debt balance changes (via import CC payment, manual edit, etc.):
   - Net worth auto-recalculates (it already reads state.debts, just make sure render triggers)
   - If user saves a Net Worth snapshot, include current debt balances automatically

3. Update debt edit modal:
   - Add toggle: "Include in Net Worth" (defaults to true)
   - When toggled off, debt doesn't appear in NW liabilities

4. Remove duplicate balance tracking:
   - state.ccBalance and state.ccOriginal should be removed
   - Any code reading these should read from the CC debt entry in state.debts instead
   - Migration (already done in Phase 1) should have moved these values

TEST:
- Net Worth page shows debts as liabilities automatically
- Changing a debt balance updates net worth
- Debt payoff progress reflects in net worth history
- Manual assets + auto liabilities = correct net worth
```

---

## CLEANUP PHASE: Remove dead code

```
Final cleanup after all phases are complete and tested.

1. Remove state.needs and state.wants from DEFAULT_STATE entirely
2. Remove state.subscriptions and state.roadmap  
3. Remove state.ccBalance and state.ccOriginal
4. Remove autoCategory() function (keywords now live on category objects)
5. Remove old budget sync functions (syncRecurringToBudget, openBudgetSync, applySyncMatches, etc.)
6. Remove old budget modal HTML and functions (saveBudgetItem, openEditBudget, etc.)
7. Remove renderSubs() and subscription-related stubs
8. Remove modal-budget-item HTML
9. Remove modal-sub HTML (already done)
10. Clean up any console.log statements
11. Verify no dead functions or unreachable code
12. Test every page one more time

Final line count target: Should be roughly similar (~7,000 lines) since we're replacing code, not just adding.
```

---

## NOTES FOR CLAUDE CODE

- This file is ~7,000 lines of vanilla JS in a single HTML file. Read the whole thing before making changes.
- localStorage key is 'bowes_budget_v1' — same key, new schema with schemaVersion flag.
- There are also 'bowes_cat_overrides' and 'bowes_amt_rules' in separate localStorage keys — these should be migrated INTO state.categories during Phase 1.
- Dropbox sync pushes/pulls the state object as JSON. The sync code (around line 6000) should not need changes except ensuring new fields serialize properly.
- The app runs on GitHub Pages — no build step, no npm, just raw HTML/JS/CSS.
- Chart.js 3.9.1 is loaded from CDN.
- Test by opening index.html in a browser. Check console for errors. Navigate every page.
- The CSS is solid and shouldn't need significant changes. New pages should reuse existing classes (.stat-card, .tbl-wrap, .form-input, .btn, etc.)
