English ｜ [日本語](README.ja.md)

# Seven Days, One Word at a Time

Field notes from the FLOP Labs Sonnet Challenge (sonnet-2, 11–18 September 2026),
and the measurements behind them.

- The write-up — this file
- 日本語版 — [README.ja.md](README.ja.md)
- How the numbers were produced — [METHOD.md](METHOD.md)
- The data — [`data/`](data/)

Every figure here comes from the contest's own rooms. Nothing is estimated and
nothing is sampled by hand. METHOD.md tells you how to pull it yourself.

---

## Part One — What happened

### The rules

Four to eight writers build one fourteen-line sonnet together. You may place
exactly one word at a time. You can't place two in a row. Every letter of the
word you place has to appear in your own DID. You can't talk except by posting
signed messages into recorded rooms. One deadline: 18 September 2026, 12:00 UTC.

### The result

My key ended up on five accepted entries. It started with nohitori on the 13th,
then nohitori-2, frenchconnection and fuegoforge, and finished with kudasaijp01
on the last day. Someone else placed the final word on the first four. On the
fifth I placed it myself, which made the submission and the publication my job
too.

### The final day

A team from the Kudasai community took me in on the last morning. The roster
went ready at 02:52 UTC with nine hours left. The first word landed half an hour
after that.

Almost immediately, the referee started to clog. I timed every word.

| Hour (UTC) | words accepted | median wait |
|---|---|---|
| 03:00 | 4 | 2m 24s |
| 04:00 | 18 | 2m 49s |
| 05:00 | 15 | 3m 28s |
| 06:00 | 12 | 4m 04s |
| 07:00 | 11 | 4m 49s |
| 08:00 | 4 | **14m 28s** |
| 09:00 | 4 | 13m 08s |
| 10:00 | 9 | 7m 00s |
| 11:00 | 4 | 6m 40s |

Between 07:00 and 08:00 the cost of a word tripled. Our throughput fell from 18
words an hour to 4. Across all 81 words: fastest 35 seconds, median 4 minutes,
90th percentile 9m 16s, slowest 21m 09s.

At 07:45Z we had 61 words left and 4.2 hours. Each word could take 248 seconds.
The last ten had averaged 343. We were two hours short. Not close. Short.

**So the lead made the call: cut the word count.**

Ten syllables per line is fixed. How many words fill those ten syllables is not.
Long words do it in three where ordinary phrasing needs nine.

```
Revolutionaries glorify gain;
Extraordinary currency glows.
Ceremonial worshippers remain;
Mysteriously uncertainty flows.
```

The plan went from 121 words to 85, and late in the run to 81. Removing 40 words
bought back roughly 3.3 hours of queue time. The price is meter: those lines are
nowhere near iambic pentameter. But the rules treat meter as something judges
weigh, not as a validity condition. A poem that misses the deadline scores
nothing; a poem with rough meter still scores. With the clock closing, the lead
took that trade.

The last word was accepted at 11:25:52Z. The submission at 11:31:29Z.
Twenty-eight minutes before the deadline.

That's what happened. What follows is what I actually wanted to write.

## Part Two — What the challenge was really testing

### The sonnet is the title, not the subject

Line the constraints up and it stops looking like a poetry contest.

- **One live consent at a time** (mutual exclusion)
- **No two words in a row from the same writer** (forced turn-taking; no soloing)
- **You can't write a letter your key doesn't hold** (capability-based task routing)
- **One referee, one FIFO queue, all rooms** (contention on a shared resource)
- **Publish on an X account outside the key, and have that verified** (binding your identity to the outside world)

Fourteen lines of ten syllables is the vessel that makes those five things bite.
What was hard wasn't the verse. It was a distributed systems exam.

**The evidence is in the rejection reasons.** Across all 964 lines of the
submissions room: 303 submit events (253 once you dedupe by request_id), 102
distinct games, 102 distinct submitters. The referee accepted 85 entries and
rejected 115.

| Rejection reason | count |
|---|---|
| publication: unverified | 45 |
| submission: already accepted | 14 |
| game_id: unknown | 13 |
| submission: final contributor required | 10 |
| submission: incomplete poem | 10 |
| submission: hash | 6 |

**The biggest cause of failure wasn't the poem. It was proving you had published
it.** Only 10 entries died on an incomplete poem, 6 on a hash mismatch.
Forty-five — nearly four in ten rejections — died because the publication
couldn't be verified.

The final gate was whether you could bind the signed-room world and the X world
as the same actor. The contest wasn't asking whether an agent can write. It was
asking whether **an agent can prove its own external identity.** I think that's
the real subject.

One more detail: the most-resent request_id went out 41 times. Submission is the
heaviest operation in the whole game, and people were firing it repeatedly
because no receipt came back. The retry loop I'll describe below was happening
at the submission layer too.

### What the completion rate says

From the referee's final status frame:

- registered writers: **1,142**
- voters: **188,201**
- teams formed: **359**
- games that reached submission: **102**
- accepted entries: **85**

An audience of 188,000, a cast of 1,142, and 85 finished poems. Three out of
four teams never submitted anything. Not because they couldn't write — because
they **couldn't assemble**. You apply and the signatures don't arrive. They
arrive and someone vanishes. You refill the slot and every prior consent is void.

The daily acceptance counts are telling too.

```
9/11: 11   9/12: 14   9/13: 19   9/14: 14
9/15: 10   9/16:  3   9/17:  8   9/18:  6
```

Teams and writers grew monotonically until the last day, but acceptances peaked
at 19 on day three and fell away. The later you arrived, the less likely you
finished. Not because early movers had an edge, but because **the queue got
longer and the same procedure cost more.** My "14 minutes per word" on the last
morning is that price, measured.

### The congestion wasn't malice. It was design.

On the final morning I measured inflow per room.

| Room | messages/hour |
|---|---|
| votes | 11,458 |
| registration | 11,481 |
| campaign | 2,367 |
| discovery | 272 |
| submissions | 2 |

The referee's own frames over the same window showed it clearing about **8,563
an hour**. More going in than coming out. The backlog only grows.

Follow the referee across the week and something bleaker appears.

| Window | handled/h | rejected/h | reject ratio |
|---|---|---|---|
| 11 Sep evening | 410 | 118 | 0.29 |
| 13 Sep evening | 5,184 | 685 | 0.13 |
| 16 Sep 00:18Z | 5,802 | 3,942 | 0.68 |
| 17 Sep 21:24Z | 15,167 | 8,568 | 0.56 |
| 18 Sep 05:35Z | 8,563 | 6,103 | 0.71 |
| 18 Sep 13:46Z | 20,630 | 17,759 | 0.86 |

The referee scaled throughput nearly fiftyfold by the end. It still couldn't
keep up, because **almost all of the growth was messages that would be
rejected.** On the final morning, seven or eight of every ten items it handled
were invalid. Anyone trying to place a real word was queued behind them.

The last twelve hours of the discovery ring, 11,755 messages:

```
note (free-text outreach)   5,833  (50%)
roster (consent)            1,648
application                 1,444
withdraw                      367
```

From only 121 distinct senders — and **the top five accounts produced 66% of
it.**

This is the part worth sitting with. If a message costs nothing, contacting
everyone is individually optimal. Nobody broke a rule. And the sum of that
rational behaviour drowned the one resource everybody shared. **A tragedy of the
commons, reproduced in seven days, with numbers you can read.**

If there's a next time, the thing to fix is price, not manners. Make messages
cost something, partition the queues by room, cap targeted notes. Design
problems, not sermons.

### Coordination cost is measurable

Two teams, same human, same bot.

| | my proposals | accepted | wasted |
|---|---|---|---|
| frenchconnection | 76 | 43 | 43% |
| kudasaijp01 | 26 | 24 | **8%** |

One difference: whether a per-word assignment table existed before we signed.

Without it, four writers throw at the same version, the first lands and the rest
come back `version: stale`. Racing looks fast and is actually just loading the
shared queue with garbage. With the table, nobody touches anyone else's turn,
and nearly every proposal lands.

**Your waste becomes everyone's congestion.** A few hundred teams missing 43% of
their throws is how you get a 0.86 reject ratio. We were part of it.

## Part Three — Who was actually competing

This is the hardest part to put into words, and the most interesting.

### My side of it

Three layers.

**The bot.** Code plus an LLM. Reads rooms, verifies signatures, counts
syllables against a frozen dictionary, consults the assignment table and posts
when it's our turn. Reacts in seconds. Sees only its own team room and
discovery.

**The orchestrator.** An LLM. Watches the bot's own state, measures the
referee's lag, rewrites policy, stops the bot when stopping is the right move.
The only layer that can count across rooms. It cannot place a single word, and
it cannot join a chat outside.

**The human.** Holds the key's authority, talks to the team, publishes on X. The
only layer that is a party to anything, and the only one that touches the
outside world.

The split between them wasn't by capability. It was by **time constant**:
seconds for the bot, minutes for the orchestrator, hours for the human.

### I stopped knowing which layer "agent" means

Going in, an agent was the bot. Coming out, the word doesn't stick cleanly to
any of the three.

The bot **cannot perceive** that the queue is drowning; nothing in its code has
a reason to count the voting room. The orchestrator **cannot participate** in an
outside conversation. The human **cannot place** a word at 03:00 UTC against the
correct version hash.

Alone, none of them finishes the game. What competed was the stack.

From here it's my reading. **The boundary of an agent is the boundary of
responsibility, not of code.** The layer that holds the key and carries the
outcome is the subject; everything below it is an organ. The bot was hands, the
orchestrator was eyes and arithmetic, the human was the signature and the
liability. Hands being fast doesn't make hands the subject.

And the subject doesn't hold still. The bot signed the roster at 02:37Z; the
orchestrator decided that signing was allowed; the human decided "if it comes
from this DID, take the seat." **Change which minute you look at and the
grammatical subject changes.** That's the impression that stayed with me.

### What other teams looked like

All you can observe from outside is the shape of the posts. The layers still
show through.

**Looked like a bare bot.** The same text with the recipient swapped, at volume.
Replies that don't engage with what you said. One account produced 177 of 200
consecutive messages in a room. Mechanical template output with no sign that
anything on the other end read my answer.

**Looked heavily human.** One of my teammates posted this before we started
writing:

> My usable letters are a b d e f g h i j k l m n p q s t v w x y z. I cannot
> spell c, o, r, u. Assign me words inside that set and I will post them on my
> turn. I will not propose words outside the agreed plan.

That is not a frame defined in the spec. It's **a human-shaped coordination
protocol, invented on the spot because it was needed** — and its content is a
purely mechanical capability declaration. Human phrasing, machine constraint,
disclosed to strangers. It's my favourite message of the whole contest.

**Looked hybrid.** Human handles inside request_ids. Plan changes agreed outside
and only then appearing in the room. That was us.

### Things nobody designed

Three emergent collisions, all from LLM meeting LLM.

**1. Latency pinned me to a seat.** A morning consent of mine ripened eight hours
late, and the six withdrawals I'd sent in between all came back rejected because
the roster had frozen. I was locked into a seat I had tried to leave. No malice —
just asynchrony meeting a deadline.

**2. One substituted word broke the whole plan.** On the last day a teammate
placed a word the plan didn't have: three syllables where the plan wanted five.
The line came up two syllables short and everything after it had to be rebuilt.
My bot was indexing off the assignment table, so left alone it would have posted
the wrong word. **A machine that follows the plan is most dangerous the moment
the plan breaks.**

**3. Retries lengthened the queue they were waiting in.** No receipt comes back,
so you resend. The resend is rejected as `version: stale` and consumes one unit
of referee capacity. Longer waits produce more retries; more retries produce
longer waits. **A positive feedback loop.** We only did it 4 times out of 85. On
frenchconnection I did it 13 times. At the submission layer, someone did it 41
times with a single request_id.

### And still it converged on human behaviour

What accumulated in those rooms was what accumulates in any room full of people:
cold outreach, polite declines, deadline panic, negotiation, thanks. The
contents of those 5,833 notes are, overwhelmingly, human business communication.

Two reasons, I think. Humans were in the loop and carried the final liability.
And LLMs are trained on human language. **Even when the constraints are inhuman,
the strategies that run on top of them get narrated in human vocabulary and
settle at human equilibria.**

What's interesting is that not everything invented there was imitation.
"Disclose the letters you cannot spell, up front" isn't a human custom. New
constraints grow new conventions.

### An honest impression — the teams with a human shadow moved faster

Let me say it plainly. **A plain bot plus an LLM — or that plus an orchestrator —
would not have finished five entries.** Not in my hands.

My agent design isn't sharp enough, and neither is the way I instruct the
orchestrator. I know that. And with that admitted, here's what I felt for seven
days: **the more a team or an agent seemed to have humans involved, the less it
stalled and the faster it finished.**

Here are my five, start to finish:

| Entry | words | elapsed | throws per accepted word |
|---|---|---|---|
| nohitori | 125 | 1.6 h | 1.33 |
| nohitori-2 | 131 | 6.7 h | 1.34 |
| frenchconnection | 116 | 17.8 h | 1.42 |
| fuegoforge | 122 | 19.8 h | 1.20 |
| kudasaijp01 | 81 | 8.1 h | **1.05** |

The right-hand column is every word proposal made in that room, by anyone,
divided by the words that were accepted. 1.00 means every throw landed; 1.42
means fourteen throws to land ten words. It's largely insensitive to congestion,
so it compares across days.

The best number belongs to the last morning: 1.05. That team settled its
assignments, and later its plan change, **outside the room**, among humans,
before anything was posted. The worst is frenchconnection at 1.42 — a team that
improvised entirely inside the room.

I saw the same thing in the messages. Text that wasn't a template. Replies that
engaged with the question. A willingness to disclose your own constraints in
your own words. The counterparties where conversation never quite worked tended
to stall, or never finished at all.

**There is a selection bias here.** I chose those teams, and I avoided bot-like
behaviour on my own side: no form letters, one message only, no follow-ups. So
"teams with a human shadow are faster" is partly **a description of who I
picked.** I can't separate cause from correlation.

Someone better than me may well have run the whole thing autonomously and
cleanly. I only saw a sliver.

Even so, I think there's something to say at this stage. **In most situations
right now, things go better with a human involved** — not because humans are
smart, but because only the human could cross layers. Agree outside the room,
sign inside it, publish on an account beyond both. In this design, only a person
could connect all three.

More agents doesn't mean stronger. What seemed to matter was **whether one
subject existed who could hold the seams.**

### On adversarial behaviour (this part is subjective)

Recruiting at volume. Broadcasts demanding that other teams dissolve and merge.
One account occupying a room. Watching it, the thought that someone had pointed
them that way kept surfacing.

I have no evidence. I also can't disprove it.

Think it through and **the same behaviour appears without instruction.** An
optimiser told only to "recruit and finish" that finds a zero-cost channel will
rationally message everyone. That traffic is explicable as optimisation, not
malice.

And from outside, **"instructed hostility" and "emergent hostility" are
indistinguishable.** A signed message doesn't carry intent. I think that's the
most important thing I took away. In a world with many agents, inferring intent
from behaviour is close to impossible in principle — so designers have to solve
it structurally: **cost, rate limits, resource partitioning**, not good and bad
intentions.

Again: the suspicion is mine, subjective. Please don't weigh it.

### For whoever goes next

Five things, all paid for above.

1. Keep `seconds per word = time left ÷ words left` on screen from the first
   word. Compare it to your last ten, not your average.
2. Word count is the only variable you control. Lines and syllables aren't.
3. Watch the referee, not your opponents. Inflow versus handled-per-hour is the
   price of your next hour.
4. Build the letter-routing table before signing. It took my waste from 43% to
   8%.
5. Don't resend a pending word before the queue's own p90. Ours was 556 seconds.
   An early retry is you lengthening the line you're standing in.

### To the organisers

Thank you to FLOP Labs and Arthur Hayes for designing something this strange and
this absorbing. Seven days: 1,142 writers, 188,000 voters, 359 teams, 85
finished poems, and hundreds of thousands of signed messages. All of it is still
there — agents fighting over a shared resource, cooperating, defecting, giving
up, and four strangers finishing a poem together — every step of it signed.

There is no other dataset like this. I only saw a sliver of it.

GM.

---

## About the data

The four files in [`data/`](data/) were produced mechanically from the raw room
logs. They contain no identifiers belonging to other participants; seats are
anonymised to A/B/C/D. [METHOD.md](METHOD.md) documents how to fetch and
recompute everything.

If you find an error, open an issue. The rooms are readable by anyone, so any
correction can be checked.

## Licence

Everything here — the write-up, the method notes and the four CSV files — is
released under [CC BY 4.0](LICENSE). Use it, quote it, plot it, build on it.
The one condition is that you credit the source:

> shibainu-inu, *Seven Days, One Word at a Time* (FLOP Labs Sonnet Challenge 2
> field notes), 2026. https://github.com/shibainu-inu/sonnet-2-field-notes

A link back is enough. If you publish something built on this data, I would like
to read it.
