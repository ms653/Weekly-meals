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

## Adding recipes without editing code

Two ways, both in the Meal Bank tab, neither needs GitHub or touching `index.html`:

- **"+ Add a recipe"** opens a form (name, dinner/breakfast, shared-prep tags, ingredients, steps). Optionally attach a photo first — it runs OCR right there in the browser (via [Tesseract.js](https://github.com/naptha/tesseract.js), no server involved) and drops the scanned text into the Ingredients box for you to tidy up and split into Steps. Saves instantly to the shared store, visible to everyone within seconds.
- **"📷 Send a photo for the weekly review"** is for when you don't want to type anything up at all — snap a cookbook page or a screenshot and send it. It's compressed and queued in the shared store, and a scheduled Claude session reviews the queue once a week: it reads each photo directly (real image understanding, not just OCR text extraction), writes a properly structured recipe into the same shared store, and clears the photo out once it's done. You'll see the new recipe appear in the Meal Bank on the next sync after that runs.

Recipes added either way show up in the Meal Bank grid and in prep clusters just like the built-in ones, marked "· added" so you can tell them apart. Assigning one to a specific day in a specific week is still a code change in `weeks` (see below) — this only covers getting the recipe itself into the bank.

## Adding a new week

1. **Add any new meals to `mealBank` first.** Give each one an id, a name, a `slot` (`'dinner'` or `'breakfast'`), and a `tags` array — reuse an existing tag (like `'sauce'`) if it shares prep with something already in the bank, so it shows up in the Meal Bank tab's clusters automatically.
2. **Add a new entry to `weeks`**, e.g. `2: { label: 'Week 2', dinners: [...], breakfasts: [...] }`, referencing meal ids from the bank by day.
3. **Add a matching entry to `shoppingLists`** with the same week number, listing categories and quantities for that week.
4. Commit the change (edit the file directly on GitHub — click the pencil icon on `index.html` — or push from your machine). Pages redeploys automatically within a minute or two.

The week picker at the top of the page updates itself from whatever numbers exist in `weeks` — no other code changes needed.

## Shopping list syncing

The shopping list (items + ticks) syncs across every device automatically, via a small free key/value store at [kvdb.io](https://kvdb.io) — no accounts, no backend to run. The page checks for changes every 8 seconds while the Shopping List tab is open, and immediately whenever you switch to that tab or bring the page back into focus. Custom recipes (see above) use the same store and the same sync pattern, on the Meal Bank tab.

The bucket ID is the `KVDB_BUCKET` constant near the top of the `<script>` tag in `index.html`. Keys currently in use in that bucket:

| Key | What it holds |
|---|---|
| `shopping-week-<n>` | That week's shopping list (items + ticks) |
| `recipes-custom` | Every recipe added via the form or the weekly review, keyed by id |
| `inbox-index` | List of photo ids waiting for the weekly review |
| `inbox-photo-<id>` | One compressed, base64-encoded photo waiting to be reviewed |

A few things worth knowing:

- There's no login on this — anyone who has the bucket ID (i.e. anyone who can read this repo's `index.html`) can read or write any of it. Fine for a household meal planner, not something to reuse for sensitive data.
- If the network is unreachable, the page falls back to the last-synced copy cached in that device's local storage (read-only until it's back online) — see the `sync-note` text under the shopping list for the current status ("Synced" / "Syncing…" / "Offline — showing your last saved copy").
- It's last-write-wins: if two people edit at the same moment, the later save overwrites the earlier one wholesale. Not an issue at the scale of a household list, but worth knowing.
- If kvdb.io ever needs to be swapped for something else (it disappears, gets slow, etc.), everything sync-related lives behind `fetchShared`/`saveShared`/`deleteShared` in `index.html` — swap those three functions for a different backend and the rest of the app doesn't change.

## The weekly recipe review

A Claude Code scheduled trigger fires once a week, and:

1. Reads `inbox-index` from the shared store — if it's empty, it does nothing.
2. For each queued photo, downloads it, looks at it directly (not OCR — genuine image understanding), and writes a structured recipe (name, dinner/breakfast, ingredients, steps, and a best guess at shared-prep tags) into `recipes-custom`, prefixed `custom-` so it can never collide with or overwrite a built-in recipe.
3. Deletes the processed photo from the store and clears it out of `inbox-index`.

It only ever talks to the shared store over HTTPS — it doesn't need a repo checkout, and it never commits to this repo. If it's ever misbehaving or you want to pause it, ask Claude to disable or delete the trigger; nothing about the rest of the app depends on it running.

## Notes

- The Meal Bank tab automatically groups any meals that share a tag into a "cluster" card, so before planning a new week you can see at a glance what's worth batch-cooking again.
- "Reset to default list" resets the *shared* list for everyone (it asks for confirmation first).
