# travel-planner

Public export of the Hermes `travel-planner` skill.

This repository contains an AI travel-planning workflow that:
- collects trip requirements conversationally
- researches Xiaohongshu travel notes with `opencli`
- synthesizes day-by-day itineraries
- geocodes places with Google Maps or AMap
- generates interactive HTML itinerary pages
- publishes trip pages to GitHub Pages

Contents:
- `SKILL.md` — the exported skill definition
- `templates/` — HTML templates for Google Maps and AMap

Notes:
- Secrets are intentionally excluded. The original local skill stores API keys in `assets/.env`; that file is not published here.
- This repo is meant as a portable/public version of the skill, not as a full runnable app by itself.
