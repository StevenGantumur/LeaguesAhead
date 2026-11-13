<h1 align="center">LeaguesAhead</h1>

<p align="center"><b>Is a ranked League of Legends game already decided by minute 15?</b><br>
LeaguesAhead pulls one player's ranked matches and their minute-by-minute timelines from the Riot API, stores them in Postgres, and serves them through FastAPI to a Next.js dashboard. The dashboard asks which early-game leads actually show up in wins, and says how far to trust each answer.</p>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.11+-3776AB?style=flat&logo=python&logoColor=white" alt="Python 3.11+">
  <img src="https://img.shields.io/badge/FastAPI-pooled-009688?style=flat&logo=fastapi&logoColor=white" alt="FastAPI">
  <img src="https://img.shields.io/badge/PostgreSQL-18-4169E1?style=flat&logo=postgresql&logoColor=white" alt="PostgreSQL 18">
  <img src="https://img.shields.io/badge/Next.js-16-555555?style=flat&logo=nextdotjs&logoColor=white" alt="Next.js 16">
  <img src="https://img.shields.io/badge/TypeScript-5-3178C6?style=flat&logo=typescript&logoColor=white" alt="TypeScript 5">
  <img src="https://img.shields.io/badge/LightGBM-analysis-9ACD32?style=flat" alt="LightGBM">
  <img src="https://img.shields.io/badge/tests-pytest-0A9EDC?style=flat&logo=pytest&logoColor=white" alt="pytest">
</p>

<p align="center"><img src="assets/media/hero.png" width="820" alt="LeaguesAhead dashboard: 193 ranked games, 99 wins and 94 losses. Ahead as a team at 15 minutes the account won 73.6% of games, behind 31.7%. Winning your own lane 63.0%, losing it 41.0%"></p>

<p align="center">Built by <a href="https://github.com/StevenGantumur">Steven Gantumur</a>. A prototype over one player's ranked games, run locally on a personal Riot key.<br>
<sub>Not endorsed by or affiliated with Riot Games. My Riot ID in the screenshots is blurred.</sub></p>

## How it works

1. **Collect.** [`collect.py`](src/collect.py) pulls match and timeline data for one account from Riot's match-v5 API. It retries on rate limits instead of skipping, and drops remakes at the door.
2. **Store.** Postgres keeps one row per game, one per player per game, and one per player per timeline frame. That last table, about 340 rows per game, holds nearly all the volume and all the predictive signal.
3. **Measure.** [`api.py`](src/api.py) compares the state of each game at 10 and 15 minutes between wins and losses, and ranks the gaps by effect size.
4. **Show.** The dashboard turns that into two questions: how often did a lead at 15 actually turn into a win, and which early leads separate wins from losses?

The screenshot at the top answers the first question. When this account's team was ahead in gold at 15 minutes, it won 73.6% of the time. Behind, 31.7%. Winning your own lane is worth far less: 63.0% against 41.0%.

| At 15 minutes | Games | Won |
|---|---|---|
| Ahead as a team | 91 | **73.6%** |
| Behind as a team | 101 | 31.7% |
| Winning your own lane | 92 | 63.0% |
| Losing your own lane | 100 | 41.0% |

A team lead swings the result by about 42 points. Winning your own lane swings it by about 22.

## The team matters more than the lane

<p align="center"><img src="assets/media/win-factors.png" width="820" alt="What shows up in wins: team gold lead at 15 minutes averages +1,855 in wins and -2,251 in losses, effect size 1.26, opened to show a tip about spending lane tempo elsewhere on the map"></p>

The second panel compares each early game metric's average in won games against lost games, and ranks them by Cohen's *d*: the gap divided by the pooled standard deviation. Raw gaps can't be compared across metrics (750 gold means nothing next to 14 CS), and standardizing puts them on one scale. Each factor opens to a tip, phrased as a lever to try rather than a rule.

Over 192 games (99W / 93L), strongest first:

| Metric | Avg in wins | Avg in losses | Cohen's *d* |
|---|---|---|---|
| **Team gold lead @15** | +1855 | −2251 | **1.26** |
| **Team XP lead @15** | +1472 | −1924 | **1.25** |
| Team gold lead @10 | +777 | −986 | 0.90 |
| Bot lane gold lead @15 | +667 | −587 | 0.78 |
| Your CS lead @15 | +8.1 | −5.8 | 0.64 |
| Support gold lead @15 | +200 | −279 | 0.63 |
| Your gold lead @15 | +316 | −436 | 0.60 |
| Top lane gold lead @15 | +417 | −551 | 0.58 |
| Jungle gold lead @15 | +289 | −438 | 0.58 |
| Mid lane gold lead @15 | +282 | −397 | 0.53 |

Team-wide gold at 15 separates wins from losses about twice as sharply as any single lane, and the player's own lane sits near the bottom of the five. That gap is large enough to take seriously. The ordering *within* the lanes is not: the standard error on *d* is about ±0.14 at this sample size, wider than the gaps between several lanes.

Every metric is measured **before minute 15**, on purpose. End-of-game stats like final gold or kills correlate enormously with winning, but that's reverse causation: you have more gold *because* you won. Sticking to the early game keeps these as leading indicators instead of restatements of the result.

## Small samples get no number at all

<p align="center"><img src="assets/media/small-sample.png" width="820" alt="Filtered to Zed, 7 games: the dashboard reports no factors and explains that at least 5 wins and 5 losses are needed"></p>

Every panel can be filtered to one champion, and that shrinks the sample fast. Below five wins and five losses, the win factors are withheld entirely, with an explanation, rather than reported as a confident-looking 0.9 built on a handful of games. Above that line, the API returns `effect_size_error` on every factor, so the error band travels with the number instead of being implied away.

## The bar a model has to clear

The notebook ([`notebooks/exploration.ipynb`](notebooks/exploration.ipynb)) predicts the result of a game from its minute-10 and minute-15 differentials. The dataset is **194 ranked games (99W / 95L) across patches 16.4 to 16.16**, and 192 of them reach minute 15.

Any accuracy figure only means something against the right baseline, and the obvious one is too easy. Always guessing the majority class gets 51.6% on those 192 games. But one rule, *predict a win whenever your team is ahead in gold at 15*, gets **70.8%**. A model that can't beat one line of SQL hasn't learned anything that table above didn't already show.

Two more caveats bound every number here. It's one player's games, so nothing generalizes to the wider player base. And it spans thirteen patches, so a random train/test split leaks later patches into training. The honest evaluation is a temporal split: train on the oldest games, test on the newest.

## What broke along the way

**The final timeline frame was being silently discarded.** Riot emits a timeline frame every 60,000 ms, plus one partial frame at the moment the game ends. The table was first keyed on `timestamp // 60000`, so for a game ending at 32:33, the 32:00 frame and the 32:33 frame both mapped to minute 32, and `ON CONFLICT DO NOTHING` threw the second away. The symptom was subtle: the longest games had exactly `max_frame + 1` rows instead of `max_frame + 2`, which looks like an off-by-one until you work out where the missing row went. The table is now keyed on the raw millisecond timestamp, with `minute` as a `GENERATED ALWAYS AS ... STORED` column, so nothing is discarded and minute-based queries still work. `test_final_partial_frame_survives` pins it.

**Riot's frame timestamps drift.** Frames are documented as arriving every 60,000 ms, but the real values are 60000, 60001, 60002 and so on. A query selecting a minute by `timestamp_ms = minute * 60000` silently returns nothing, which is exactly what happened when the stats query was first written. Selection now goes through the derived `minute` column, deduplicated with `DISTINCT ON (match_id, puuid, minute)` ordered by `timestamp_ms`, since a game ending mid-minute contributes a second, partial frame in that minute.

**Remakes had to be excluded at three layers.** A remake, an early surrender in the first few minutes, looks like a loss with every player at 0/0/0. It drags down loss-side averages and pads the match list with games nobody played. Remakes are skipped at collection via `gameEndedInEarlySurrender`, filtered out of `/matches` by `game_duration`, and removed from already-collected data by [`SQL/migrations/002_delete_remakes.sql`](SQL/migrations/002_delete_remakes.sql).

**Rate limiting is a retry, not a skip.** The original error handler caught every HTTP error the same way and moved on, so a 429 silently cost a game. `riotGet` now reads the `Retry-After` header, sleeps and retries the same request. A related bug: the `continue` in the exception handler skipped the sleep between requests, so the request right after a rate-limit error fired with no delay at all, precisely when backing off matters most. The sleep now lives in a `finally` block.

**Connections leaked on errors.** The API opened a connection per request and closed it after the query, so any exception raised straight past the `close()`. Connections now come from a `ThreadedConnectionPool` created at startup and go back through a context manager's `finally`.

**`ORDER BY match_id` was accidentally correct.** Match IDs sort chronologically today because they're sequential per region, but that's a property of Riot's ID assignment, not a guarantee. `/matches` joins `matches` and orders by `game_creation`.

**`participants(puuid)` needed its own index.** The primary key is `(match_id, puuid)`, and a composite B-tree is ordered by its leading column, so it can't serve the `WHERE puuid = ?` lookup `/matches` runs on every request.

## System design

```
┌──────────────┐  match-v5   ┌──────────────┐   rows    ┌──────────────┐
│  Riot API    │ ──────────> │  collect.py  │ ────────> │  PostgreSQL  │
└──────────────┘  timelines  │  429 retry,  │           │  4 tables    │
                             │  remake skip │           └──────┬───────┘
                             └──────────────┘                  │
                                      ┌────────────────────────┴──┐
                                      v                           v
                             ┌──────────────────┐        ┌──────────────────┐
                             │  api.py          │        │  exploration     │
                             │  FastAPI, pooled │        │  .ipynb, pandas, │
                             │  connections     │        │  LightGBM        │
                             └────────┬─────────┘        └──────────────────┘
                                      │ JSON
                                      v
                             ┌──────────────────┐
                             │  Next.js 16      │
                             │  dashboard       │
                             └──────────────────┘
```

Collection runs by hand, not on a schedule, because a personal Riot key expires every 24 hours (unfortunately). Everything downstream of Postgres only reads.

| Table | One row per | Notes |
|---|---|---|
| `matches` | game | Creation time, duration, patch, queue and winning team |
| `participants` | player per game, 10 per match | End-of-game stats, role, champion |
| `participant_timelines` | player per timeline frame, about 340 per game | Keyed on `(match_id, puuid, timestamp_ms)`, with `minute` generated from the timestamp |
| `players` | account | Cached Riot ID. Match data carries only PUUIDs, and names are mutable, so it's keyed on PUUID |

| Endpoint | Returns |
|---|---|
| `GET /health` | Liveness, including a real database round trip |
| `GET /player` | The tracked account's Riot ID and overall record |
| `GET /matches?limit=&champion_id=` | Recent matches, newest first |
| `GET /stats/champions` | Per-champion record, KDA, CS and average game length |
| `GET /stats/conversion` | How often a lead at 15 minutes became a win |
| `GET /stats/win-factors?minutes=&champion_id=` | Early-game metrics ranked by how strongly they separate wins from losses |

Every stats endpoint accepts `champion_id` to narrow the analysis to one champion.

## Stack

| What | How |
|---|---|
| Collection | Python, `requests`, Riot match-v5 and account-v1 |
| Storage | PostgreSQL 18, `psycopg2` |
| API | FastAPI and `uvicorn`, pooled connections |
| Frontend | Next.js 16, React 19, TypeScript, Tailwind 4, champion names from Data Dragon |
| Analysis | pandas, LightGBM, scikit-learn |
| Tests | pytest |

## Run it

Needs Python 3.11+, Node 20+ and PostgreSQL.

```bash
python -m venv venv && source venv/Scripts/activate   # macOS/Linux: source venv/bin/activate
pip install -r requirements.txt -r requirements-dev.txt
cp .env.example .env                                   # then fill in the values below
```

| Variable | What it is |
|---|---|
| `RIOT_API_KEY` | From [developer.riotgames.com](https://developer.riotgames.com/). Personal keys expire every 24 hours |
| `PUUID` | The account to track, from account-v1: `/riot/account/v1/accounts/by-riot-id/{gameName}/{tagLine}` |
| `DATABASE_URL` | `postgresql://user:password@host:port/database` |
| `MATCH_TARGET` | Optional. How many matches to collect, default 200 |

**Percent-encode special characters in the database password.** A literal `/` makes libpq think the host ends there, which produces a confusing "could not translate host name" error. Write it as `%2F`.

```bash
createdb leaguetracker
psql -d leaguetracker -f SQL/schema.sql

python src/collect.py          # 200 matches by default, about 2 requests every 5 seconds
python src/collect.py 250      # or pick a number
```

Then start both halves:

```bash
# terminal 1, API on http://localhost:8000
uvicorn src.api:app --reload

# terminal 2, dashboard on http://localhost:3000
cd frontend && cp .env.local.example .env.local && npm install && npm run dev
```

The migrations in [`SQL/migrations/`](SQL/migrations/) are only for databases created before a schema change. A fresh database from `schema.sql` doesn't need them.

## Tests

```bash
python -m pytest tests/ -q
```

The suite covers the three pure functions that turn Riot responses into database rows. It needs no network and no database, so it runs anywhere. The fixtures in `tests/fixtures/` follow the match-v5 schema, and `python tests/capture_fixture.py <match_id>` replaces them with a real captured match.

## Project structure

| Folder | What's in it |
|---|---|
| [`src/`](src/) | `collect.py`, the Riot API client and ingestion loop, and `api.py`, the FastAPI read API |
| [`SQL/`](SQL/) | `schema.sql` for a fresh database, and ordered migrations for older ones |
| [`frontend/`](frontend/) | The Next.js dashboard |
| [`notebooks/`](notebooks/) | Exploratory analysis and the win-prediction model |
| [`tests/`](tests/) | Unit tests for the row-building transforms, with fixtures |
| [`assets/media/`](assets/media/) | README screenshots |

## What it is and isn't

**It is** an end-to-end pipeline and the reasoning behind it: collection against a rate-limited third-party API, a schema that survived contact with real data, and analysis that states its own uncertainty instead of overselling a small sample.

**It isn't** a product, yet:
- One player's games. Nothing here generalizes to the wider player base.
- No sign-up and no player search. The tracked account is fixed by `PUUID` in the environment.
- Personal Riot keys expire daily, so collection is a manual run rather than a scheduled job.
- It runs locally. There is no deployment.

## Where this is going

The full plan is in [TODO.md](TODO.md). In short:

1. **One-command setup.** A `docker-compose.yml` with Postgres and the API, with the schema applied on first boot and seed data, so trying it takes `docker compose up` and a Riot key.
2. **Hosted.** Managed Postgres, the API on a container host, the frontend on Vercel. Collection then has to become a scheduled job, which needs a Production Riot key.
3. **Any account, not just one.** Search by Riot ID, collect on demand with a job queue, and share one rate-limit budget across users. This also needs a Production key, which means passing Riot's review.

## Credits

Match and timeline data from the [Riot Games API](https://developer.riotgames.com/). Champion names from [Data Dragon](https://developer.riotgames.com/docs/lol#data-dragon). Built with [FastAPI](https://fastapi.tiangolo.com), [PostgreSQL](https://www.postgresql.org), [Next.js](https://nextjs.org), [LightGBM](https://lightgbm.readthedocs.io) and [pandas](https://pandas.pydata.org).

<sub>LeaguesAhead isn't endorsed by Riot Games and doesn't reflect the views or opinions of Riot Games or anyone officially involved in producing or managing Riot Games properties. Riot Games, and all associated properties are trademarks or registered trademarks of Riot Games, Inc.</sub>
