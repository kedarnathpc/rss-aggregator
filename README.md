# RSS Feed Aggregator
This is a Go Language learning project following along with [Free Code Camp's Intro To Go Course](https://www.youtube.com/watch?v=un6ZyFkqFKo&pp=ygUaZnJlZSBjb2RlIGNhbXAgaW50cm8gdG8gZ28%3D) with [Lane Wager](https://github.com/wagslane)

RSS (Really Simple Syndication) is basically a web feed that allows access to updates on a certain website in computer-readable format (typically XML)

# Setup

### Setting Up Environment Variables

```bash
go get github.com/joho/godotenv
```

Which just installs the dependency godotenv, which allows for the reading of the
.env file.

Additionally, after installation, he invokes:

```bash
go mod vendor
```

Which creates a `vendor` directory which keeps track of the installed dependencies.

```bash
go mod tidy
```

Cleans up imports so that they are seen by our editor/project. You may need to
run vendor again after running tidy:

```bash
go mod vendor
```

### Setting Up The Router

Install a commonly used go router package called chi:

```bash
go get github.com/go-chi/chi
```

And also the cors package from the same repo maintainer:

```bash
go get github.com/go-chi/cors
```

**Goose**

Goose is sets up migrations using .sql files. You'll need to create a database
in your local postgres instance called `rssagg`:

```Postgres
CREATE DATABASE rssagg;
```

Once created, within the `sql/schema` directory, create a `001_users.sql` file
and insert the following:

```sql

-- +goose Up

CREATE TABLE users (
        id UUID PRIMARY KEY,
        created_at TIMESTAMP NOT NULL,
        updated_at TIMESTAMP NOT NULL,
        name TEXT NOT NULL
);

-- +goose Down

DROP TABLE users;
```

Within your env sample file insert your credentials for your local postgres
database (see env.sample).

Using this DB_URL, you can now migrate your database up from within the
`sql/schema` directory run:

```bash
goose postgres $DB_URL up
```

Within psql or pgcli clients, you can now view your created table from within
the rssagg database like so:

```sql
use rssagg;
\d;
\d users;
```

You can also run the following to clean up your database.

```bash
goose postgres $DB_URL down
```

**SQLC**

Now that you have goose and sqlc installed, you can use sqlc to generate your
queries. sqlc requires a configuration yaml file in the root of your project
. Here's what we have in ours:

```yaml
version: "2"
sql:
  - schema: "sql/schema"
    queries: "sql/queries"
    engine: "postgresql"
    gen:
      go:
        out: "internal/database"
```

We now create our Queries within the sql/queries directory, creating .sql files,
here is our users.sql file:

```sql
-- name: CreateUser :one

INSERT INTO users (id, created_at, updated_at, name)
VALUES ($1, $2, $3, $4) RETURNING *;
```

Note the comments at the top, these are required and basically create an sql
function called CreateUser that is called once. Ther est of the sql statement
should be somewhat familiar to you.

You'll need to run the sqlc command from the root of your project (where your
sqlc.yaml file is) like so:

```bash
sqlc generate
```

It then utilizes the sqlc yaml file to generate the sql queries you needed by
reading the passed files (in this case the schema/001_users.sql file and the
queries/user.sql file). It will then generate a series of go files in the
internal/database directory.

We'll now adjust our DB_URL env variable to disable sslmode (and remove the
rssagg db specification) giving us something like this:

```
DB_URL=postgres://postgres:password@localhost:5432/rssagg?sslmode=disable
```

You'll also need to import a driver called `pq` from github, even though it is
never called, this allows the line:

```go
type apiConfig struct {
	DB *database.Queries
}
```

To work. Install it using go get:

```bash
go get github.com/lib/pq
```

And in your main.go file, import it:

```go
import (
    _ "github.com/lib/pq"
)
```

You'll also need to point the database to the sqlc generated files by importing
them into your main.go file:

```go
	"github.com/kedarnathpc/rss-aggregator"
```

## Building And Running The Server

```bash
go build && ./rssagg
```

## Quering The DB From pgcli:

```pgcli
use rssagg;
select * from users;
```

## Routes

### Health Check
- **GET /v1/healthz**
  - **Description**: Check the health status of the API.
  - **Response**: `200 OK` if the service is running.

### Error Simulation
- **GET /v1/err**
  - **Description**: Simulate an error for testing purposes.
  - **Response**: `500 Internal Server Error`

### Users
- **POST /v1/users**
  - **Description**: Create a new user.
  - **Request Body**: JSON object with user details.
  - **Response**: `201 Created` if the user is successfully created.

- **GET /v1/users**
  - **Description**: Get details of the authenticated user.
  - **Headers**: 
    - `Authorization: Bearer <token>`
  - **Response**: `200 OK` with user details.

### Feeds
- **POST /v1/feeds**
  - **Description**: Create a new feed.
  - **Headers**: 
    - `Authorization: Bearer <token>`
  - **Request Body**: JSON object with feed details.
  - **Response**: `201 Created` if the feed is successfully created.

- **GET /v1/feeds**
  - **Description**: Get a list of all feeds.
  - **Response**: `200 OK` with a list of feeds.

### Posts
- **GET /v1/posts**
  - **Description**: Get posts for the authenticated user.
  - **Headers**: 
    - `Authorization: Bearer <token>`
  - **Response**: `200 OK` with a list of posts.

### Feed Follows
- **POST /v1/feed_follows**
  - **Description**: Follow a feed.
  - **Headers**: 
    - `Authorization: Bearer <token>`
  - **Request Body**: JSON object with feed follow details.
  - **Response**: `201 Created` if the feed follow is successfully created.

- **GET /v1/feed_follows**
  - **Description**: Get the list of feeds followed by the authenticated user.
  - **Headers**: 
    - `Authorization: Bearer <token>`
  - **Response**: `200 OK` with a list of followed feeds.

- **DELETE /v1/feed_follows/{feedFollowID}**
  - **Description**: Unfollow a feed.
  - **Headers**: 
    - `Authorization: Bearer <token>`
  - **Path Parameters**: 
    - `feedFollowID` - ID of the feed follow to be deleted.
  - **Response**: `204 No Content` if the feed follow is successfully deleted.

## Authentication
Some routes require the user to be authenticated. Use the `Authorization` header with a bearer token to access these routes.

## CORS
The API supports Cross-Origin Resource Sharing (CORS) with the following configuration:
- **Allowed Origins**: `https://*`, `http://*`
- **Allowed Methods**: `GET`, `POST`, `PUT`, `DELETE`, `OPTIONS`
- **Allowed Headers**: `Accept`, `Authorization`, `Content-Type`, `X-CSRF-Token`
- **Exposed Headers**: `Link`
- **Allow Credentials**: `false`
- **Max Age**: `300`

## Environment Variables
Ensure to set the following environment variables in your `.env` file:
- `PORT`: Port number on which the server will run.
- `DB_URL`: Database URL for PostgreSQL connection.

## How It Works

1. **Feed Fetching**: The scraper fetches a batch of feeds to process.
2. **Concurrent Processing**: It uses multiple goroutines to process each feed concurrently.
3. **Database Operations**: Each feed's posts are parsed and stored in the database.
4. **Periodic Execution**: The process repeats at regular intervals specified in the configuration.

## Logic code
### StartScraping Function
This function initializes the scraping process. It sets up a ticker to trigger scraping at regular intervals and uses goroutines for concurrent processing.
```
func startScraping(
	db *database.Queries,
	concurrency int,
	timeBetweenRequests time.Duration,
) {
	log.Printf("Scraping on %v goroutines every %s duration", concurrency, timeBetweenRequests)
	ticker := time.NewTicker(timeBetweenRequests)
	for ; ; <-ticker.C {
		feeds, err := db.GetNextFeedsToFetch(
			context.Background(),
			int32(concurrency),
		)
		if err != nil {
			log.Printf("Error getting next feeds to fetch: %v", err)
			continue
		}

		wg := &sync.WaitGroup{}
		for _, feed := range feeds {
			wg.Add(1)
			go scrapeFeed(db, wg, feed)
		}
		wg.Wait()
	}
}

```

### ScrapeFeed Function
This function handles the actual fetching and processing of a single feed. It marks the feed as fetched, retrieves the feed data, parses it, and stores the posts in the database.
```
func scrapeFeed(db *database.Queries, wg *sync.WaitGroup, feed database.Feed) {
	defer wg.Done()

	_, err := db.MarkFeedAsFetched(context.Background(), feed.ID)
	if err != nil {
		log.Printf("Error marking feed as fetched: %v", err)
		return
	}

	rssFeed, err := urlToFeed(feed.Url)
	if err != nil {
		log.Printf("Error fetching feed: %v", err)
		return
	}
	for _, item := range rssFeed.Channel.Item {
		description := sql.NullString{}
		if item.Description != "" {
			description.String = item.Description
			description.Valid = true
		}

		pubAt, err := time.Parse(time.RFC1123Z, item.PubDate)
		if err != nil {
			log.Printf("couldn't parse date %v with err %v", item.PubDate, err)
		}
		_, err = db.CreatePost(context.Background(),
			database.CreatePostParams{
				ID:          uuid.New(),
				CreatedAt:   time.Now().UTC(),
				UpdatedAt:   time.Now().UTC(),
				Title:       item.Title,
				Description: description,
				PublishedAt: pubAt,
				Url:         item.Link,
				FeedID:      feed.ID,
			})
		if err != nil {
			if strings.Contains(err.Error(), "duplicate key") {
				continue
			}
			log.Printf("Error creating post: %v", err)
			return
		}
	}

	log.Printf("Feed %s collected, %v posts found", feed.Name, len(rssFeed.Channel.Item))
}

```

Link to a sample RSS https://www.wagslane.dev/index.xml

## Tools used:

1. Routing: Chi Router https://github.com/go-chi/chi
2. Database: Postgresql
3. Goose: Databse migrations https://github.com/pressly/goose
4. SQLC: Generate type-safe code from raw SQL https://github.com/sqlc-dev/sqlc
5. Thunder Client: API Testing

