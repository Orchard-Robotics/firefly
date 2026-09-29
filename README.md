# Firefly Concert

Build an HTTP server for a shared "light wall". Clients `POST` bursts of energy onto rectangles
of the wall. Other clients `GET` the whole wall to display it. State lives in memory in one
process. The challenge is correct shared state under concurrent requests, and speed.

## The wall

- A wall is a 64 × 64 grid of integers. Every cell starts at 0.
- `(0,0)` is the top-left cell. `x` goes right, `y` goes down.
- A wall has a version. It starts at 0 and goes up by exactly 1 on every successful burst.
- A burst adds `energy` to every cell in a rectangle. Values only add. No cap, no wrap.
- A snapshot lists all cells as one flat array of 4,096 integers, row by row: `index = y * 64 + x`.
- Walls are independent. Changing one never changes another.

![The 64 by 64 wall with its four corners labelled, and a zoom of the top-left 8 by 8 showing each cell's index](docs/images/coordinates.png)

## Server requirements

- Start command takes the port as an argument: `<your command> --port 8765`
- Listen on `127.0.0.1`. HTTP/1.1 with keep-alive connections.
- Every response has `Content-Type: application/json` and a correct `Content-Length` (or chunked encoding).
- Response bodies contain exactly the fields shown below. Do not wrap them in another object.
- Request bodies are JSON objects. Unknown fields are ignored. Key order and whitespace do not matter.
- Restarting the server clears all walls.

---

# Routes

| Method | Path | What it does |
|---|---|---|
| `POST` | `/walls` | Create a wall |
| `POST` | `/walls/{wall_id}/bursts` | Add energy to a rectangle |
| `GET` | `/walls/{wall_id}` | Read the whole wall |

`wall_id` is 1 to 64 characters from `A-Z a-z 0-9 _ -`. Same rule in a body and in a path.

## Errors shared by all routes

| Status | Body | When |
|---|---|---|
| `404` | `{"error":"not_found"}` | Unknown route, or the wall in the path does not exist |
| `400` | `{"error":"invalid_request"}` | Body is not a JSON object, a field is missing, wrong type, out of range, or the rectangle does not fit |
| `409` | `{"error":"already_exists"}` | Create with an ID that already exists |

Check in this order: **404, then 400, then 409.** A missing wall is 404 even if the body is
garbage. A rejected request changes nothing: no pixels, no version.

Numeric fields must be JSON integers. `1` is valid. `1.0`, `1e0`, `"1"`, `true`, `null` are not.

---

## `POST /walls`

### What it does

Creates an empty wall with the given ID. All cells 0, version 0.

### Request

```json
{"wall_id":"main-stage"}
```

| Field | Type | Rule |
|---|---|---|
| `wall_id` | string | 1 to 64 characters from `A-Z a-z 0-9 _ -` |

### Returns

`201 Created`

```json
{"wall_id":"main-stage","version":0}
```

### Errors

| Status | Body | When |
|---|---|---|
| `400` | `{"error":"invalid_request"}` | Body is not a JSON object, or `wall_id` is missing or invalid |
| `409` | `{"error":"already_exists"}` | The ID already exists. Leave the existing wall untouched |

### Under concurrency

Eight clients create the same ID at the same instant. Exactly one gets `201`. Seven get `409`.

---

## `POST /walls/{wall_id}/bursts`

### What it does

Adds `energy` to every cell in the rectangle and increments the wall's version. Both happen as
one atomic step.

### Request

```json
{"x":2,"y":3,"width":2,"height":2,"energy":5}
```

| Field | Type | Range | Meaning |
|---|---|---|---|
| `x` | integer | 0 to 63 | Left column of the rectangle |
| `y` | integer | 0 to 63 | Top row of the rectangle |
| `width` | integer | 1 to 64 | Columns covered |
| `height` | integer | 1 to 64 | Rows covered |
| `energy` | integer | 1 to 100 | Added to every covered cell |

All five fields are required.

The rectangle must fit on the wall: `x + width <= 64` and `y + height <= 64`. Reject one that
hangs over the edge. Do not clip it.

A burst covers every cell where `x <= cell_x < x + width` and `y <= cell_y < y + height`.
The example above covers exactly `(2,3)`, `(3,3)`, `(2,4)`, `(3,4)`.

### Returns

`200 OK`

```json
{"version":1}
```

`version` is the number this burst was assigned. If N bursts succeed on a new wall, they return
versions 1 through N, each exactly once. Responses may arrive out of order. Identical requests
sent twice are two bursts.

### Errors

| Status | Body | When |
|---|---|---|
| `404` | `{"error":"not_found"}` | The wall does not exist. Checked before the body is looked at |
| `400` | `{"error":"invalid_request"}` | Bad JSON, missing field, non-integer, out of range, or rectangle does not fit |

```text
{"x":63,"y":63,"width":2,"height":2,"energy":1}   -> 400   x + width = 65 > 64
{"x":2,"y":3,"width":2.0,"height":2,"energy":5}   -> 400   2.0 is not an integer
{"x":2,"y":3,"width":2,"height":2}                -> 400   energy missing
{"x":2,"y":3,"width":2,"height":2,"energy":101}   -> 400   energy above 100
POST /walls/no-such-wall/bursts  (any body)       -> 404   wall check comes first
```

![Left: a 4 by 4 burst at x=60,y=60 fits and is applied. Right: a 2 by 2 burst at x=63,y=63 crosses the edge and is rejected, leaving the wall unchanged](docs/images/out-of-bounds.png)

---

## `GET /walls/{wall_id}`

### What it does

Returns the wall's version and all 4,096 cells, taken from one consistent state.

### Request

No body.

### Returns

`200 OK`

```json
{"version":2,"pixels":[0,0,0,0, ... 4096 integers ... ,0,0]}
```

| Field | Type | Meaning |
|---|---|---|
| `version` | integer | Current version of the wall |
| `pixels` | array of exactly 4,096 integers | Every cell, row by row: `index = y * 64 + x`. Include the zeros |

`pixels` is one flat array. Not 64 nested arrays, not a sparse map, not an image, not a sum.

### Errors

| Status | Body | When |
|---|---|---|
| `404` | `{"error":"not_found"}` | The wall does not exist |

### Consistency rules

- `version` and `pixels` come from the same state. A concurrent burst is either fully in the
  snapshot or fully out of it. Never half.
- A read that starts after a burst's `200` response must include that burst.
- One client's sequential reads never see the version go backward.

![Three snapshots while a burst is in flight: before the burst is allowed, after the burst is allowed, half-applied is never allowed](docs/images/snapshot-consistency.png)

---

# Worked example

Fresh wall `demo`:

| Step | Request | Response |
|---|---|---|
| 1 | `POST /walls` `{"wall_id":"demo"}` | `201` `{"wall_id":"demo","version":0}` |
| 2 | `POST /walls/demo/bursts` `{"x":2,"y":3,"width":2,"height":2,"energy":5}` | `200` `{"version":1}` |
| 3 | `POST /walls/demo/bursts` `{"x":3,"y":3,"width":1,"height":1,"energy":2}` | `200` `{"version":2}` |
| 4 | `GET /walls/demo` | `200`, version 2, pixels below |
| 5 | `POST /walls/demo/bursts` `{"x":63,"y":63,"width":2,"height":2,"energy":1}` | `400` `{"error":"invalid_request"}` |
| 6 | `GET /walls/demo` | Same as step 4 |

![The worked example step by step: empty wall, then the 2 by 2 burst of 5, then the 1 by 1 burst of 2 on top](docs/images/worked-example.png)

Top-left corner after step 4. Everything else is 0.

```text
       x:  0  1  2  3  4  5  6  7
  y=3      0  0  5  7  0  0  0  0      pixels[194]=5   pixels[195]=7
  y=4      0  0  5  5  0  0  0  0      pixels[258]=5   pixels[259]=5
```

Sum of all pixels: 22. The full JSON is in `example_snapshot.json`.

---

# Scoring

- **70 points correctness.** Exact output, validation, creation races, concurrent snapshots, final state.
- **30 points speed.** Only awarded if every correctness check passes, including under load.
- The grader sends 10,000 requests per second for 3 seconds per workload over 64 keep-alive
  connections. 75% bursts, 25% reads. Two workloads: one shared wall, then four independent walls.
- Full speed points: keep up with that rate with read and write p99 at or under 10 ms.
  The load phase runs 30 times and the speed score is the average.

```sh
firefly-grader --url http://127.0.0.1:8765 --correctness-only
firefly-grader --url http://127.0.0.1:8765 --runs 1     # quick speed check
firefly-grader --url http://127.0.0.1:8765              # scored: 30 runs
```

# Ground rules

- 90 minutes to build, 20 minutes to discuss.
- Any language, runtime, framework. The choice is yours to make and defend. It affects both
  how hard correctness is and how many speed points you can reach.
- An AI coding assistant may write code. You must choose the stack, design the synchronization,
  run the grader, find the bottleneck, and write `NOTES.md` yourself.
- Submit: source, start command, the assistant transcript, and `NOTES.md` covering your stack
  choice, synchronization design, the bottleneck you found, one optimization you measured, and
  which code the assistant wrote.
