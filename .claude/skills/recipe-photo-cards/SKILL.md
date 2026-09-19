---
name: recipe-photo-cards
description: Use when the user shares one or more photos of recipes (cookbook pages, screenshots, handwritten cards) in this repo and asks to turn them into recipe cards, add them to the Meal Bank, or build "a version of" recipe-cards.html. Extracts each photo into a standalone Cook-Mode-styled HTML page matching this repo's existing recipe-cards.html.
---

# Recipe photo → Cook Mode card page

Turns a batch of shared recipe photos into a standalone HTML page styled exactly
like `recipe-cards.html` in this repo: tappable recipe tiles that open a "Cook
Mode" modal (ingredients + utensils screen first, then step-through cooking
steps with Back/Next, progress dots, swipe support). That file — not this
skill — is the source of truth for the design. Read it fresh each run rather
than trusting a remembered copy of its CSS/JS, since it may have changed.

## When to run this

The user shares photos and asks for recipe cards, asks to add recipes to the
Meal Bank via photos, or asks for "a version of" / "another one like"
`recipe-cards.html`. If they instead want a quick single recipe added to the
live app's Meal Bank (the kvdb-backed one, via the "+ Add a recipe" form or
the weekly photo review), that's a different, already-built flow — see this
repo's `README.md` ("Adding recipes without editing code" and "The weekly
recipe review"). This skill is specifically for producing a new *card page*
like `recipe-cards.html`.

## Step 1: Clarify scope if it's ambiguous

- **New page or extend an existing one?** If the photos are clearly a new,
  different batch/theme from what's already in `recipe-cards.html` (or any
  other card page already in the repo), default to creating a **new page**
  rather than appending forever to one file — ask the user for a short
  theme/title if it's not obvious from the photos (e.g. "Weeknight Pasta
  Cards"), and derive a kebab-case filename from it (e.g.
  `weeknight-pasta-cards.html`). If the user explicitly says to add these to
  an existing page instead, do that.
- Confirm you're not about to overwrite work — check the target filename
  isn't already a different, unrelated page before writing.

## Step 2: Extract each photo into the data schema

Look at each photo directly (vision, not OCR — you can read layout and
structure a plain OCR pass would mangle). For each recipe, extract into this
exact shape (matches the `RECIPES` array in `recipe-cards.html`):

```js
{
  id: 'kebab-case-slug',              // stable, unique within this page
  name: 'Recipe Name',                // HTML-escape & (-> &amp;) etc.
  sub: 'one-line subtitle',
  difficulty: 'Easy' | 'Medium' | 'Hard',
  timeActive: '10 min',               // prep/active time as written
  timeTotal: '3–6 hrs',               // total/cook time as written
  servings: '2 adults + 2 littles',   // whatever the source actually says
  ingredients: [
    { icon:'chicken', name:'Chicken thighs, boneless & skinless', qty:'500g', allergen:'Contains: Milk' } // allergen optional, omit the field entirely if none
  ],
  notIncluded: 'black pepper',        // pantry staples not listed as ingredients; omit field if none
  utensils: [ { label:'Slow cooker' }, { label:'Frying pan' } ],  // no icon needed, see below
  steps: [
    { title:'Two to four words', body:"Imperative instructions with <b>ingredient</b> bolded the moment it's used.", callout:{ kind:'tip'|'important', text:'...' } } // callout optional, omit unless it's genuinely grounded in the source text
  ]
}
```

Rules, matching how the existing 7 recipes were done — keep new ones consistent:

- **Never fabricate nutrition/calorie data.** There's no real per-ingredient
  macro database behind this page (unlike the real Plateful app), so don't
  add a calories stat or nutrition table with invented numbers.
- **Allergens**: only add an `allergen` field when it's directly obvious from
  the literal ingredient (dairy → Milk, wheat products → Wheat, Gluten, stock
  cubes → Celery is standard for almost all UK stock cubes, Worcestershire
  sauce → Fish from anchovies, coconut → Coconut per UK allergen labelling).
  Don't do a full allergen audit or guess at "may contain" cross-contamination
  risks beyond what's stated on the packet norms above.
- **Callouts**: TIP (green) or IMPORTANT (red) only where the source text
  actually has one — a real food-safety note, a genuine failure mode, or a
  nice-to-know the original recipe calls out. Most steps have none. Don't
  invent callouts to fill space.
- **Step titles**: 2-4 words, sentence case, the action not "Step 3".
- **Difficulty**: infer reasonably (a one-pot dump-and-cook is Easy; multiple
  components or techniques bump it up) — don't default everything to Easy
  without thinking about it.

## Step 3: Icons

`recipe-cards.html`'s icon sprite reuses exact icon SVGs from
[ms653/plateful](https://github.com/ms653/plateful)'s real deployed app
(`meal-builder-prototype.html`), not invented ones, for ingredients it already
has: **chicken, beef, onion, garlic, carrot, potato, cheese, pasta, lentils,
chickpeas, coconutmilk, spinach, bread, corn, tomato**. Copy those `<symbol>`
defs verbatim from `recipe-cards.html`'s `<svg><defs>` block (or from the
`ms653/plateful` repo directly if you have it checked out) — don't redraw
them from scratch.

For anything else, `recipe-cards.html` already has hand-drawn custom icons in
the same flat-shape style (**lamb, celery, squash, courgette, cream, milk,
butter, canned, flour, spice, herb, stockcube, oil, sauce, raisin**) — reuse
those too if the ingredient matches. Only draw a genuinely new icon (same
style: `viewBox="0 0 64 64"`, 2-4 flat shapes, one accent highlight, no
gradients, a faint ellipse "shadow" at the base like the real ones) if none
of the existing icons fit. **Utensils don't get icons** in the real app's
Cook Mode — they're rendered as a plain comma-separated text line
("You'll also need: X, Y"), not icon pills. Don't add utensil icons.

## Step 4: Build the page

Read `recipe-cards.html` in full and copy its structure for the new page:
the `:root` tokens (`--accent:#17C299` etc.), Space Grotesk + Inter fonts,
the `.rcard`/`.card-grid` tile styles, and the entire Cook Mode overlay
(`#cookOverlay`, `renderCook`, `cookNext`/`cookPrev`/`cookReady`, the swipe
handler, the step -1 "ingredients screen" pattern, done-state dots, "Done
cooking" on the last step). Swap in the new `RECIPES` array and intro
copy/title; keep everything else — this is a *shared design system*, not a
one-off, so don't drift the CSS or Cook Mode behaviour between pages.

Include a shopping list banner + modal too, in the same style, if the new
page has more than one or two recipes. Just total up quantities as given —
don't do a full ingredient-sharing rejig (swapping stock cube types, sharing
a squash between recipes, etc.) unless the user separately asks for that,
the way it was done for the original 7.

**Known bug class to avoid**: when building the shopping-list modal's
per-category `data-shop-list` lookup, match sections by **array index**, not
by the category name string. A category name containing `&` (e.g. "Fruit &
Veg") gets HTML-entity-escaped for display but decoded in the live DOM, so a
`querySelector` built from the raw JS string silently fails to match and
throws — this exact bug shipped once already in `recipe-cards.html` and broke
the whole modal. Index-based lookup sidesteps it entirely.

## Step 5: Validate before shipping

Don't just eyeball it — this file's interactivity (modal open, tab/step
switching, checkbox re-render) needs a real DOM to catch runtime errors; a
syntax check alone isn't enough (see the bug above, which was valid JS that
still threw at runtime). In a scratch directory:

```bash
npm install jsdom --silent   # registry.npmjs.org is reachable even though
                              # most other domains aren't in this environment
```

Then load the page with `runScripts: 'dangerously'`, dispatch a click on
each `data-id` "Cook with me" button and on the shopping-list open button,
and assert no `window.onerror` fired and the modal actually populated
(non-trivial `innerHTML` length, expected row/step counts). Fix anything it
catches before proceeding.

## Step 6: Wire it up and ship

- If it's a new page, add a link to it somewhere discoverable — e.g. a tile
  in `index.html`'s Meal Bank actions (matching the existing "📋 Printable
  recipe cards" tile pattern) or a link from `recipe-cards.html` itself if
  there's now more than one card page. Ask the user if it's not obvious
  where they'd want it linked from.
- Run a quick `node -e "new Function(...)"` syntax check on the final file
  (cheap, catches typos the jsdom pass might not exercise) alongside the
  jsdom smoke test above.
- Commit with a plain, descriptive message and push to `main` — no PR
  needed for this repo's own static pages, matching how the rest of this
  repo's content changes ship.
