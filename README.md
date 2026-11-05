# rssagg

An RSS feed aggregator and REST API written in Go. Users register, add RSS feeds, and follow the feeds they care about. A background scraper fetches every feed on a schedule and stores new posts in PostgreSQL.

## Features

- User registration with auto-generated API keys
- API key authentication for protected endpoints
- Add, list, follow, and unfollow RSS feeds
- Concurrent background scraper (goroutines + `sync.WaitGroup`) that fetches the least recently updated feeds first
- Posts saved to PostgreSQL, with duplicate posts ignored by URL
- Type-safe database access generated with sqlc
- Schema migrations managed with goose

## Tech Stack

- **Language:** Go
- **Router:** [chi](https://github.com/go-chi/chi) with [cors](https://github.com/go-chi/cors)
- **Database:** PostgreSQL via [lib/pq](https://github.com/lib/pq)
- **Query generation:** [sqlc](https://sqlc.dev)
- **Migrations:** [goose](https://github.com/pressly/goose)
- **Config:** [godotenv](https://github.com/joho/godotenv)

## Prerequisites

- Go (a recent version)
- PostgreSQL
- [goose](https://github.com/pressly/goose): `go install github.com/pressly/goose/v3/cmd/goose@latest`
- [sqlc](https://sqlc.dev) (only needed if you change the SQL): `go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest`

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/Danbaba1/rssagg.git
cd rssagg
```

### 2. Create the database

```bash
psql -U postgres -c "CREATE DATABASE rssagg;"
```

### 3. Configure environment variables

Create a `.env` file in the project root:

```env
PORT=8000
DB_URL=postgres://postgres:YOUR_PASSWORD@localhost:5432/rssagg?sslmode=disable
```

`.env` is listed in `.gitignore`, so your credentials are not committed.

### 4. Run the migrations

```bash
cd sql/schema
goose postgres "postgres://postgres:YOUR_PASSWORD@localhost:5432/rssagg?sslmode=disable" up
cd ../..
```

### 5. Build and run

```bash
go build
./rssagg
```

On Windows PowerShell use `./rssagg` (or `.\rssagg.exe`). You should see:

```
Server starting on port 8000
Scraping on 10 goroutines every 1m0s duration
```

## API Reference

Base URL: `http://localhost:8000/v1`

Protected endpoints require your API key in the `Authorization` header:

```
Authorization: ApiKey YOUR_API_KEY
```

| Method | Endpoint | Auth | Description |
| ------ | -------- | ---- | ----------- |
| GET | `/healthz` | No | Readiness check |
| GET | `/err` | No | Returns a sample error response |
| POST | `/users` | No | Create a user and receive an API key |
| GET | `/users` | Yes | Get the authenticated user |
| POST | `/feeds` | Yes | Add a new feed |
| GET | `/feeds` | No | List all feeds |
| POST | `/feed_follows` | Yes | Follow a feed |
| GET | `/feed_follows` | Yes | List feeds the user follows |
| DELETE | `/feed_follows/{feedFollowID}` | Yes | Unfollow a feed |

### Examples

**Create a user**

```bash
curl -X POST http://localhost:8000/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Daniel"}'
```

**Add a feed**

```bash
curl -X POST http://localhost:8000/v1/feeds \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Dan on Dev.to", "url": "https://dev.to/feed/danbaba1"}'
```

**Follow a feed**

```bash
curl -X POST http://localhost:8000/v1/feed_follows \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"feed_id": "FEED_UUID"}'
```

## How the Scraper Works

1. On startup, `startScraping` runs in its own goroutine and ticks once per minute.
2. Each tick it asks the database for the next feeds to fetch (oldest `last_fetched_at` first).
3. Each feed is fetched concurrently in its own goroutine and marked as fetched.
4. The RSS XML is parsed and every item is saved as a post. Posts whose URL already exists are skipped.

Feed URLs must point to an RSS/XML feed, not a regular web page.

## Project Structure

```
rssagg/
├── main.go               # Server setup, routes, starts the scraper
├── scraper.go            # Background scraping loop
├── rss.go                # Fetches and parses RSS XML
├── models.go             # API response models and DB-to-API converters
├── json.go               # JSON response helpers
├── handler_*.go          # HTTP handlers
├── middleware_auth.go    # API key authentication
├── sqlc.yaml             # sqlc configuration
├── sql/
│   ├── schema/           # goose migrations
│   └── queries/          # SQL queries used by sqlc
└── internal/database/    # Generated sqlc code
```

## Regenerating Database Code

After editing files in `sql/queries` or `sql/schema`:

```bash
sqlc generate
```

## Author

Daniel Oladepo, [@Danbaba1](https://github.com/Danbaba1)
