# Recipe inbox

Drop a photo of a cookbook page or a screenshot of a recipe here (any filename, any of jpg/jpeg/png/heic/webp) and it'll be reviewed and turned into a recipe automatically within about a week.

The easiest way in: tap "📷 Send a photo for the weekly review" on the Meal Bank tab of [the app itself](https://ms653.github.io/Weekly-meals/) — it opens this folder's upload page directly. You'll need a free GitHub account and to be added as a collaborator on this repo first (ask whoever set this up).

A scheduled Claude session checks this folder once a week, reads each photo, writes a proper recipe into `weekly-recipes.json` at the repo root, and removes the photo from here. This file itself never gets removed - it's just here so the folder isn't empty.
