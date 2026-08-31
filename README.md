# This Week's Kitchen

A single-file family meal planner: weekly plan, shopping list, and a growing bank of meals tagged by shared prep (meat sauce, mash, sausages) so future weeks can reuse batch-cooking opportunities.

Everything lives in one file — `index.html` — so there's nothing to build or install.

## Get it live on GitHub Pages

You don't need git installed for this — the GitHub website handles it.

1. Go to **github.com/new** and create a new repository (e.g. `family-meal-planner`). Keep it **Public** if you want a shareable link, or Private if it's just for you (Pages works either way on a free personal account, but private repos need Pages enabled per-repo under Settings → Pages).
2. On the new repo's page, click **"uploading an existing file"** (or Add file → Upload files).
3. Drag in `index.html` and `README.md` from this folder, and commit.
4. Go to **Settings → Pages** in the repo.
5. Under **Build and deployment → Source**, choose **Deploy from a branch**.
6. Branch: **main**, folder: **/ (root)** → **Save**.
7. Wait about a minute, then refresh — GitHub shows the live URL at the top of that Pages settings section (something like `https://yourusername.github.io/family-meal-planner/`).

That URL is shareable with Mel, or anyone else — no login needed to view it.

### If you'd rather use the terminal (you already have GitHub CLI set up)

```bash
cd family-meal-planner
gh repo create family-meal-planner --public --source=. --remote=origin --push
gh api repos/:owner/family-meal-planner/pages -X POST -f source[branch]=main -f source[path]=/
```

## How the data is organised

Everything editable lives near the top of the `<script>` tag in `index.html`:

- **`mealBank`** — every meal that's ever been cooked, tagged with any shared prep (`sauce`, `mash`, `sausage`). This is the growing library.
- **`weeks`** — which meals from the bank are assigned to which day, per week number.
- **`shoppingLists`** — one shopping list per week number, with quantities already worked out for what's shared.
- **`tagMeta`** — the three shared-prep tags and their colours/labels. Add a new one here if a new kind of shared prep shows up (e.g. `roast-veg`).

## Adding a new week

1. **Add any new meals to `mealBank` first.** Give each one an id, a name, a `slot` (`'dinner'` or `'breakfast'`), and a `tags` array — reuse an existing tag (like `'sauce'`) if it shares prep with something already in the bank, so it shows up in the Meal Bank tab's clusters automatically.
2. **Add a new entry to `weeks`**, e.g. `2: { label: 'Week 2', dinners: [...], breakfasts: [...] }`, referencing meal ids from the bank by day.
3. **Add a matching entry to `shoppingLists`** with the same week number, listing categories and quantities for that week.
4. Commit the change (edit the file directly on GitHub — click the pencil icon on `index.html` — or push from your machine). Pages redeploys automatically within a minute or two.

The week picker at the top of the page updates itself from whatever numbers exist in `weeks` — no other code changes needed.

## Shopping list syncing

The shopping list (items + ticks) now syncs across every device automatically, via a small free key/value store at [kvdb.io](https://kvdb.io) — no accounts, no backend to run. The page checks for changes every 8 seconds while the Shopping List tab is open, and immediately whenever you switch to that tab or bring the page back into focus.

The bucket ID is the `KVDB_BUCKET` constant near the top of the `<script>` tag in `index.html`. A few things worth knowing:

- There's no login on this — anyone who has the bucket ID (i.e. anyone who can read this repo's `index.html`) can read or write the list. Fine for a household grocery list, not something to reuse for sensitive data.
- If the network is unreachable, the page falls back to the last-synced copy cached in that device's local storage (read-only until it's back online) — see the `sync-note` text under the list for the current status ("Synced" / "Syncing…" / "Offline — showing your last saved copy").
- It's last-write-wins: if two people edit at the same moment, the later save overwrites the earlier one wholesale. Not an issue at the scale of a household list, but worth knowing.
- If kvdb.io ever needs to be swapped for something else (it disappears, gets slow, etc.), everything sync-related lives behind `fetchShared`/`saveShared` in `index.html` — swap those two functions for a different backend and the rest of the app doesn't change.

## Notes

- The Meal Bank tab automatically groups any meals that share a tag into a "cluster" card, so before planning a new week you can see at a glance what's worth batch-cooking again.
- "Reset to default list" resets the *shared* list for everyone (it asks for confirmation first).
