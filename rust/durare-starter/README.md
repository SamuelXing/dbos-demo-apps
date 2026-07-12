# durare Starter

A DBOS starter app on [**durare**](https://crates.io/crates/durare) — a
DBOS-compatible durable-execution SDK for Rust. It is a port of the
[Go starter](../../golang/dbos-go-starter) and demonstrates the same features:
durable workflows and steps, durable sleep, cron schedules, queues, events, and
human-in-the-loop messaging — behind a small [`axum`](https://crates.io/crates/axum)
web server that reuses the Go starter's frontend verbatim.

To learn more about programming with durare, see the
[crate docs](https://docs.rs/durare) (start with the `durability` and
`determinism` guides).

## Setup

1. **Rust** — a recent stable toolchain (durare's MSRV is 1.88).

2. **Postgres** — durare stores workflow state in Postgres. Any local instance
   works; for example, with Docker:

   ```bash
   docker run --rm -d --name durare-pg -p 5432:5432 \
     -e POSTGRES_PASSWORD=dbos postgres:16
   createdb -h localhost -U postgres durare_starter   # password: dbos
   ```

3. **Connection string** — point the app at your database. It reads
   `DBOS_SYSTEM_DATABASE_URL` (falling back to `DATABASE_URL`):

   ```bash
   export DBOS_SYSTEM_DATABASE_URL="postgres://postgres:dbos@localhost:5432/durare_starter"
   ```

   durare creates its `dbos` system schema and tables automatically on first run.

## Running

```bash
cargo run
```

The app serves at <http://localhost:8080>, with the DBOS admin server on
<http://localhost:3001>.

## What it demonstrates

- **Workflows and steps** — `ExampleWorkflow` runs three durable steps and
  publishes a progress event after each. Watch it live, then hit **Crash the
  app** mid-run: on restart the workflow *resumes from its last checkpoint* —
  completed steps are served from the log and never re-run.
- **Scheduled workflows** — a cron workflow you can apply, pause, resume, and
  trigger at runtime via the managed-schedule API.
- **Queues** — enqueue work onto a durable queue with a bounded worker
  concurrency.
- **Communication** — a human-in-the-loop workflow that durably waits (up to
  120s) for an approval message before deciding whether to continue.

## HTTP endpoints

| Method + path | What it does |
| --- | --- |
| `GET /` | the frontend |
| `GET /workflow/:id` | start `ExampleWorkflow` with the given id |
| `GET /last_step/:id` | the workflow's progress (0–3), read from its event |
| `POST /crash` | exit the process — for demonstrating recovery |
| `GET /schedule/status` | cron, schedule state, recent run counts |
| `POST /schedule/apply` \| `/pause` \| `/resume` \| `/trigger` | manage the schedule |
| `GET /queue/status` | worker concurrency and recent run counts |
| `POST /queue/enqueue` | enqueue an `EnqueuedWorkflow` |
| `POST /queue/concurrency` | (see note below) |
| `POST /comm/start` | start a `CommunicationWorkflow`, returns its id |
| `GET /comm/status/:id` | the workflow's current state |
| `POST /comm/approve/:id` \| `/comm/deny/:id` | send the decision |

## Notes for durare (vs. the Go starter)

The port is faithful, with two differences that reflect durare's design:

- **Recovery is an explicit call.** After `launch()`, the app calls
  `engine.recover()` to resume workflows a previous run left unfinished (durare
  keeps launch and recovery separate). This is what makes the **Crash the app**
  demo resume rather than strand its workflow.
- **Queue configuration is sealed at launch.** durare's `register_queue` takes
  `&mut self` and the engine is shared `&self` afterward, so worker concurrency
  is set once, before `launch()` — it cannot be retuned at runtime the way the
  Go starter's slider does. The `/queue/concurrency` endpoint therefore reports
  the value fixed at startup; changing it means restarting with a new
  `register_queue` call.
