# Are You Deadlocked?

ClemTracker is a single-page web app that answers one question about a [Deadlock](https://store.steampowered.com/app/1422450/Deadlock/) player: are their stats slipping?

Enter a Steam ID and the page gives two verdicts:

- **Recently?** YES if performance has dropped over the player's latest games.
- **Overall?** YES if performance is declining across every game on record.

Below the verdicts, it shows where the player is improving or regressing, their strengths and weaknesses, and how they performed in each match.

Not affiliated with Valve. Match data comes from the community-run [Deadlock API](https://deadlock-api.com).

## Features

- **Staged verdict reveal:** each answer appears in turn, and the animation is skipped for people who have reduced motion turned on.
- **Area trends:** Winning, Fighting, Survival, Economy, Farming and Teamplay are each marked Improving, Regressing or Steady.
- **Stat-by-stat tiles:** recent and all-games movement for eight stats, coloured only when a move is outside normal swings.
- **Strengths and weaknesses:** the last 50 games compared with players at the same rank on the same heroes, plus best and weakest hero.
- **Match by match:** a chart of the last 40 games, and a table with each game's score and its best and worst stat.
- **Trend, hero and rank history:** a chart for each key stat, a per-hero table and a rank timeline.
- **Recent lookups:** up to 12 players are remembered in your browser for quick switching.
- **Example mode:** "See an example" shows sample data without needing a Steam ID.

## Running locally

The app is one HTML file with no dependencies to install and no build step. From the repo folder, run:

```bash
python -m http.server 8765 --bind 127.0.0.1
```

Then open <http://127.0.0.1:8765/index.html>. Any static file server works in place of Python.

You need an internet connection, because the page loads its data from the Deadlock API and its fonts from Google Fonts.

### Looking up a player

The search box accepts:

| Input | Example |
| --- | --- |
| SteamID64 | `76561198068963539` |
| Deadlock account ID | `108697811` |
| Steam3 ID | `[U:1:108697811]` |
| Steam profile URL | `https://steamcommunity.com/profiles/76561198068963539` |
| Custom profile URL or player name | `https://steamcommunity.com/id/name` |

Name searches use the first close match, so a SteamID64 is the most reliable option.

The page reloads your last lookup automatically. To get back to the landing page, run `localStorage.clear()` in the browser console and refresh.

## How the verdicts work

### Match score

Every game gets a match score, where **50 is the player's own average** and 65 is one standard deviation above it. Each stat is compared with the player's own history, then grouped into six areas that count equally:

| Area | Stats |
| --- | --- |
| Winning | Win rate |
| Fighting | Kills, KDA |
| Survival | Deaths (fewer is better) |
| Economy | Souls per minute |
| Farming | Last hits per minute, denies |
| Teamplay | Assists |

### Recently?

Compares the last 20 games with the 20 before them, using Welch's t-test on match scores. The window can be changed to 10, 30 or 50 games. If the player has too few games to fill both windows, their history is split in half.

### Overall?

Fits a least-squares trend line through the match score of every game. The answer is YES only if the slope is significant and the line falls at least 3 points, because very long histories can make tiny slopes look significant.

### What counts as YES

A drop has to be bigger than normal game-to-game swings would explain, at about 95% one-sided confidence. A small dip still gets NO, with a line explaining why. Players with fewer than 10 games get a `?`.

In simulated players, stable players received a false YES about 5% of the time. A sharp recent slump was caught about 91% of the time, a milder one about half the time, and a long decline every time.

### Which games count

Ranked and unranked matches count. Private lobbies, bot games and other non-standard modes are always excluded. Street Brawl is excluded unless "Include Street Brawl" is ticked.

### Limitations

- Scores are relative to the player's own history, so climbing into tougher lobbies, switching to a support hero or a big patch can all look like decline.
- The rank comparison falls back to all players when the account has no usable rank. Beating that average is easier than beating your own rank's.
- Last hits per minute in the rank comparison is an estimate, because the API doesn't provide that average directly.
- The API only has the matches it has been able to fetch. Players who run the [Deadlock API ingest tool](https://github.com/deadlock-api/deadlock-api-ingest), or whose friends do, get the most complete history.

## Configuration

The verdict settings are constants near the top of the script in `index.html`:

| Constant | Default | Purpose |
| --- | --- | --- |
| `T_CRIT` | `1.645` | t-statistic a move must reach to count. Lower it for more sensitive verdicts. |
| `MIN_GAMES` | `10` | Fewest games needed for a YES or NO. |
| `MIN_TREND_PTS` | `3` | Score points the Overall trend line must fall. |
| `SCORE_SD` | `15` | Spread of the match score around 50. |
| `BENCH_GAMES` | `50` | Recent games compared with the rank average. |
| `BENCH_GAP` | `0.1` | Gap from the rank average, in single-match standard deviations, that counts as a strength or weakness. |

## How it's built

Everything lives in `index.html`: plain HTML, CSS and JavaScript, with charts drawn as hand-written SVG. There is no framework, backend or account system.

The page calls these endpoints on `https://api.deadlock-api.com` directly from the browser:

| Endpoint | Used for |
| --- | --- |
| `/v1/players/{account_id}/match-history` | Match stats and rank history |
| `/v1/players/{account_id}/rank` | Current rank |
| `/v1/players/steam` | Name, avatar and profile link |
| `/v1/players/steam-search` | Finding a player by name |
| `/v1/analytics/player-stats/metrics` | Rank-bracket averages per hero |
| `/v1/assets/heroes`, `/v1/assets/ranks` | Hero and rank names and images |

Recent lookups and tracking snapshots are stored in the browser's `localStorage` under `dlp.*` keys. They never leave the browser. The only other outside requests are for Google Fonts and for the hero, rank and Steam avatar images.

## Deployment

The app is a static file, so it can be hosted on any static host, such as GitHub Pages, by serving `index.html`.
