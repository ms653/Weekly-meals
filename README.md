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

Two ways, both starting from the Meal Bank tab:

- **"+ Add a recipe"** — no GitHub, no account. Opens a form (name, dinner/breakfast, shared-prep tags, ingredients, steps). Optionally attach a photo first — it runs OCR right there in the browser (via [Tesseract.js](https://github.com/naptha/tesseract.js), no server involved) and drops the scanned text into the Ingredients box for you to tidy up and split into Steps. Saves instantly to the shared store, visible to everyone within seconds.
- **"📷 Send a photo for the weekly review"** is for when you don't want to type anything up at all — it opens GitHub's upload page for this repo's `recipe-inbox/` folder. Drop a cookbook photo or screenshot there (needs a free GitHub account, and being added as a collaborator on this repo — a one-time setup, ask whoever set this up). A scheduled Claude session checks that folder once a week: it looks at each photo directly (real image understanding, not just OCR text extraction), commits a properly structured recipe into `weekly-recipes.json`, and removes the photo. The new recipe shows up in the Meal Bank the next time the page loads after that runs.

Recipes added either way show up in the Meal Bank grid and in prep clusters just like the built-in ones, marked "· added" so you can tell them apart. Assigning one to a specific day in a specific week is still a code change in `weeks` (see below) — this only covers getting the recipe itself into the bank.

## Adding a new week

1. **Add any new meals to `mealBank` first.** Give each one an id, a name, a `slot` (`'dinner'` or `'breakfast'`), and a `tags` array — reuse an existing tag (like `'sauce'`) if it shares prep with something already in the bank, so it shows up in the Meal Bank tab's clusters automatically.
2. **Add a new entry to `weeks`**, e.g. `2: { label: 'Week 2', dinners: [...], breakfasts: [...] }`, referencing meal ids from the bank by day.
3. **Add a matching entry to `shoppingLists`** with the same week number, listing categories and quantities for that week.
4. Commit the change (edit the file directly on GitHub — click the pencil icon on `index.html` — or push from your machine). Pages redeploys automatically within a minute or two.

The week picker at the top of the page updates itself from whatever numbers exist in `weeks` — no other code changes needed.

## Shopping list syncing

The shopping list (items + ticks) syncs across every device automatically, via a small free key/value store at [kvdb.io](https://kvdb.io) — no accounts, no backend to run. The page checks for changes every 8 seconds while the Shopping List tab is open, and immediately whenever you switch to that tab or bring the page back into focus. Recipes added via the "+ Add a recipe" form (see above) use the same store and the same sync pattern, on the Meal Bank tab.

The bucket ID is the `KVDB_BUCKET` constant near the top of the `<script>` tag in `index.html`. Keys currently in use in that bucket:

| Key | What it holds |
|---|---|
| `shopping-week-<n>` | That week's shopping list (items + ticks) |
| `recipes-custom` | Recipes added via the "+ Add a recipe" form, keyed by id |

A few things worth knowing:

- There's no login on this — anyone who has the bucket ID (i.e. anyone who can read this repo's `index.html`) can read or write either of these. Fine for a household meal planner, not something to reuse for sensitive data.
- If the network is unreachable, the page falls back to the last-synced copy cached in that device's local storage (read-only until it's back online) — see the `sync-note` text under the shopping list for the current status ("Synced" / "Syncing…" / "Offline — showing your last saved copy").
- It's last-write-wins: if two people edit at the same moment, the later save overwrites the earlier one wholesale. Not an issue at the scale of a household list, but worth knowing.
- If kvdb.io ever needs to be swapped for something else (it disappears, gets slow, etc.), everything sync-related lives behind `fetchShared`/`saveShared` in `index.html` — swap those two functions for a different backend and the rest of the app doesn't change.

## The weekly recipe review

Recipes from photos deliberately **don't** go through kvdb.io — the scheduled Claude session that reviews them runs in an environment whose network access is locked to GitHub (plus a couple of package registries), so it can't reach a third-party keystore at all. Everything for this feature routes through the repo instead:

1. A photo lands in `recipe-inbox/` (uploaded via GitHub's web UI — see above).
2. Once a week, a Claude Code scheduled trigger fires, checks that folder — if it's empty, it does nothing.
3. For each photo, it looks at it directly (not OCR — genuine image understanding) and writes a structured recipe (name, dinner/breakfast, ingredients, steps, and a best guess at shared-prep tags) into `weekly-recipes.json` at the repo root, under an id prefixed `reviewed-` so it can never collide with a built-in recipe or one added through the form.
4. It deletes the processed photo and pushes everything as one commit directly to `main`.

`index.html` fetches `weekly-recipes.json` (a same-origin request, since it's served right alongside the page) on load and merges it in with the built-ins and the kvdb-backed custom recipes — see `allRecipes()`.

If it's ever misbehaving or you want to pause it, ask Claude to disable or delete the trigger; nothing about the rest of the app depends on it running. If you ever want to review a photo yourself without waiting for the schedule, just add the recipe to `weekly-recipes.json` (or use the "+ Add a recipe" form) and delete the photo from `recipe-inbox/`.

## Notes

- The Meal Bank tab automatically groups any meals that share a tag into a "cluster" card, so before planning a new week you can see at a glance what's worth batch-cooking again.
- "Reset to default list" resets the *shared* list for everyone (it asks for confirmation first).
