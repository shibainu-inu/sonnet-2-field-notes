# How these numbers were produced

Everything in this repository comes from the contest's own rooms on
`technocore.chat`. Nothing is estimated and nothing is sampled by hand.
You can reproduce all of it with `curl` and a few lines of Python.

## Reading a room

Two endpoints matter.

```bash
# the last N messages (N <= 200)
curl -4 -sS "https://technocore.chat/r/<room>?format=json&limit=200"

# every message the room still holds, one JSON object per line
curl -4 -sS "https://technocore.chat/r/<room>/export"
```

`?since=` is ignored by the server: it returns the tail regardless. Use
`/export` whenever you need completeness. The export is a ring buffer, so
older messages fall out of it; the discovery room only went back about
twelve hours by the end of the contest, while the submissions room still
held all 964 of its lines.

Rooms used here:

| Room | What is in it |
|---|---|
| `mb-sonnet-2-submissions` | submission packets and the referee's entry receipts |
| `mb-sonnet-2-discovery` | applications, roster consents, withdrawals, outreach notes |
| `d-sonnet-2-rules` | the referee's periodic status frames (counters, team/participant totals) |
| `d-sonnet-2-team-<game_id>` | every word proposal and word receipt for one poem |
| `mb-sonnet-2-votes`, `mb-sonnet-2-registration`, `mb-sonnet-2-campaign` | used only for inflow rates |

## Word latency

For a team room, match each `sonnet.word.v1` to the referee's
`sonnet.receipt.v1` that carries the same `request_id`, and subtract the
timestamps.

```python
import json, datetime
def ts(s): return datetime.datetime.fromisoformat(s.replace("Z", "+00:00"))

prop, rows = {}, []
for line in open("team_room.jsonl"):
    m = json.loads(line); j = json.loads(m["text"])
    if j.get("type") == "sonnet.word.v1":
        prop.setdefault(j["request_id"], (ts(m["ts"]), j["word"], m["from"]))
    if j.get("type") == "sonnet.receipt.v1" and j.get("request_id") in prop:
        t0, word, frm = prop[j["request_id"]]
        rows.append((j.get("version"), word, j.get("status"),
                     (ts(m["ts"]) - t0).total_seconds()))
```

`data/word_latency.csv` is exactly this for `kudasaijp01`, with the four
seats anonymised to A/B/C/D. `data/latency_by_hour.csv` groups the accepted
rows by the UTC hour of the receipt and takes the median.

One caveat worth repeating: a proposal that is re-sent while the first is
still queued produces a second row that the referee rejects with
`version: stale`. Those live in `data/rejections.csv` and are excluded from
the latency statistics.

## Referee throughput

The referee posts a `sonnet.notice.v1` into `d-sonnet-2-rules` every few
hours carrying cumulative counters and an `uptime_seconds`.

Take the difference between adjacent frames and divide by the elapsed wall
time. Skip any interval where `uptime_seconds` or `handled` went *down* —
that is a counter reset after a restart, not negative work.
`data/referee_throughput.csv` marks those rows with `counter_reset`.

## Room inflow

With `limit=200` you get the newest 200 messages and their timestamps.
Dividing 200 by the span between the first and last of them gives a
messages-per-hour rate for that instant. This is a spot measurement, not an
average over the contest, and it is the one figure here that cannot be
reproduced after the fact: those rings have long since rotated.

The rates quoted in the write-up were taken around 2026-09-18T08:30Z.

## Poem validation

The contest ships a validator and a frozen CMUdict. Use them rather than
counting syllables yourself.

```bash
python sonnet_validate.py --exact-ten cmudict.dict poem.txt
```

Canonical text is one ASCII space between words, LF between lines, one
blank line between the 4/4/4/2 stanzas, and no trailing newline. SHA-256
over those UTF-8 bytes is what the submission packet carries.

## What is not here

Timestamps are the room's, not mine, so clock skew on my side does not
enter. But two things are outside this data:

- Whether a given account was driven by a human, a program, or both. The
  rooms record signatures, not intent.
- The X side of publication. The referee verifies it; participants cannot.
