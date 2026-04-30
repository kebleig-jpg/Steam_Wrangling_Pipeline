# Steam Gaming Data Pipeline

A Python data pipeline that collects, integrates, and cleans gaming behavior data from the **Steam Web API** and **SteamSpy API**. Built for DATA 304 (Data Wrangling) at the University of Tennessee.

## Table of Contents

- [Project Overview](#project-overview)
- [Data Sources](#data-sources)
- [Pipeline Architecture](#pipeline-architecture)
- [Wrangling Methods](#wrangling-methods)
- [Output Files](#output-files)
- [Setup and Reproduction](#setup-and-reproduction)
- [Version History](#version-history)
- [Known Limitations](#known-limitations)
- [AI Disclosure](#ai-disclosure)
- [Contributors](#contributors)

---

## Project Overview

This project builds an analysis-ready dataset of Steam gaming behavior by pulling data from two independent APIs, merging them on shared keys, and applying eight distinct wrangling methods across five versioned iterations. The pipeline starts with two seed users (Matt and Cody), discovers up to 200 additional users from their friend networks, and produces a clean, normalized dataset suitable for downstream analysis.

## Data Sources

| Source | Endpoint | What It Provides |
|--------|----------|------------------|
| Steam Web API | `GetPlayerSummaries/v2` | Username, country, account creation date |
| Steam Web API | `GetOwnedGames/v1` | Game library with playtime per title |
| Steam Web API | `GetPlayerAchievements/v1` | Achievement completion counts per game |
| Steam Web API | `GetFriendList/v1` | Friend network for user discovery |
| SteamSpy API | `appdetails` | Genre, tags, owner estimates, ratings, pricing |

All data is collected via authenticated REST API calls using personal Steam API keys. SteamSpy data is publicly available and does not require authentication.

## Pipeline Architecture

```
Seed Users (Matt & Cody)
        │
        ▼
Friend Discovery (depth=1, cap=200)
        │
        ▼
┌───────────────────────────────────────┐
│  Per-User Data Collection             │
│  ├── Player Summaries (ISteamUser)    │
│  ├── Owned Games (IPlayerService)     │
│  ├── Achievements (ISteamUserStats)   │
│  └── SteamSpy Metadata (per appid)   │
└───────────────────────────────────────┘
        │
        ▼
    Merge on steamid + appid
        │
        ▼
    5-Stage Wrangling Pipeline
        │
        ▼
    Clean Dataset + Genre Lookup Table
```

## Wrangling Methods

The pipeline applies eight required wrangling methods across five sequential batches. Each batch reads the previous version's CSV and writes a new one, creating a traceable transformation history.

### Batch 1 → `steam_v1_raw.csv`
**Raw data collection only.** No wrangling applied. Serves as the baseline for before/after comparison.

### Batch 2 → `steam_v2_typed.csv`

| Method | Implementation | Example |
|--------|---------------|---------|
| **Type Conversion** | Cast ratings, price, achievements, and discount fields from object/float to proper int/float using `pd.to_numeric()` and `.astype()` | `achievements_completed`: float → int |
| **Date Parsing** | Converted Unix timestamp (`timecreated`) to datetime via `pd.to_datetime(unit='s')`. Derived `account_age_years` from the parsed date. | `1608912819` → `2020-12-25 18:33:39` → `4.3 years` |

### Batch 3 → `steam_v3_cleaned.csv`

| Method | Implementation | Example |
|--------|---------------|---------|
| **Regex** | Extracted numeric lower bound from owner range strings using `str.extract(r"(\d+)")` after stripping commas | `"20,000,000 .. 50,000,000"` → `20000000.0` |
| **Text Cleaning** | Stripped whitespace, removed trademark symbols (™®©) via regex, normalized multiple spaces. Flattened tag dictionaries to top-3 comma-separated strings. Filled null values in country and genre with `"Unknown"`. | `"Valheim™ "` → `"Valheim"` |

### Batch 4 → `steam_v4_deduped.csv`

| Method | Implementation | Example |
|--------|---------------|---------|
| **Fuzzy Matching** | Compared game names from Steam API (`name`) vs. SteamSpy (`steamspy_name`) using `fuzz.ratio()` from the `thefuzz` library. Flagged pairs scoring below 90%. Retained Steam API name as canonical. | `"PUBG: BATTLEGROUNDS"` vs `"PUBG: BATTLEGROUNDS"` → score 100 |
| **Deduplication** | Dropped duplicate rows on the composite key `(steamid, appid)` using `drop_duplicates()` | Rows before vs. after logged |

### Batch 5 → `steam_v5_final.csv`

| Method | Implementation | Example |
|--------|---------------|---------|
| **Reshaping** | Split comma-separated genre strings into individual rows using `str.split(", ").explode()`, producing a long-format dataset with one row per game-genre pair | `"Action, RPG, Indie"` → 3 rows |
| **Tidying** | Created a normalized genre lookup table (`steam_genre_lookup.csv`) where each row represents one game-genre observation. Master table remains wide (one row per user-game) for general analysis. | Lookup table: `appid × single_genre` |

## Output Files

| File | Description | Format |
|------|-------------|--------|
| `steam_v1_raw.csv` | Raw collected data, no wrangling | Wide, one row per user-game |
| `steam_v2_typed.csv` | After type conversion + date parsing | Wide |
| `steam_v3_cleaned.csv` | After regex + text cleaning | Wide |
| `steam_v4_deduped.csv` | After fuzzy matching + deduplication | Wide |
| `steam_v5_final.csv` | Final clean master dataset | Wide, one row per user-game |
| `steam_genre_lookup.csv` | Tidy genre reference table | Long, one row per game-genre |
| `steam_genres_long.csv` | Full exploded dataset for genre analysis | Long, one row per user-game-genre |

## Setup and Reproduction

### Requirements

```
python >= 3.8
pandas
requests
thefuzz
```

### Steps

1. Clone the repository
2. Open the notebook in Google Colab (recommended) or Jupyter
3. Obtain a Steam Web API key from [https://steamcommunity.com/dev/apikey](https://steamcommunity.com/dev/apikey)
4. Update the `users` list in Batch 1 with your API key(s) and Steam ID(s)
5. Run all cells sequentially from top to bottom
6. Output CSVs are saved to the configured `output_dir` path

### Runtime Notes

- Full pipeline (2 seed users + 200 friends) takes approximately 1–3 hours depending on library sizes
- Checkpoints are saved every 25 users to prevent data loss on disconnection
- A keep-alive JavaScript cell is included to prevent Colab session timeouts
- SteamSpy is rate-limited to ~1 request/second; the pipeline includes `time.sleep()` delays

## Version History

| Commit | Message |
|--------|---------|
| v1 | Raw data collection from Steam Web API and SteamSpy |
| v2 | Type conversion + date parsing |
| v3 | Regex + text cleaning |
| v4 | Fuzzy matching + deduplication |
| v5 | Reshaping + tidying + final export |

Each commit modifies the same notebook file. Git diff history shows exactly which cells were added at each stage.

## Known Limitations

- **Private profiles**: Users with private Steam profiles return no game or friend data and are silently skipped
- **Achievement gaps**: Games without achievement support return no data from the achievements endpoint, resulting in null values filled with 0
- **Owner estimates**: SteamSpy provides owner counts as ranges (e.g., "20,000,000 .. 50,000,000"), not exact figures. The pipeline extracts only the lower bound.
- **Rate limiting**: SteamSpy enforces undocumented rate limits. Aggressive parallelization will result in blocked requests.
- **Temporal snapshot**: All data reflects the state at the time of collection. Playtime, ratings, and prices change continuously.
- **Friend network bias**: Discovered users share a social connection with the seed users, introducing selection bias. Results are not representative of the global Steam population.

## AI Disclosure

AI tools (Claude by Anthropic) were used during development to assist with code formatting, structuring the wrangling pipeline, and drafting documentation. **All final evaluations, corrections, data validation, and analytical decisions were made by the group members.** The AI did not have access to the Steam API keys or any collected data.

## Contributors

- Cody Kihlberg
- Matt Shaaya
- Kebba Leigh

DATA 304 — Data Wrangling | University of Tennessee | Spring 2025