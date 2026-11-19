# Beli-style rankings: spec

`movie_beli` is a pure-Python, stateless library for personal movie rankings
in the style of the Beli restaurant app. It has no UI and no server: a host
app (the hosted page lives in
[hansonhqma/reel-club](https://github.com/hansonhqma/reel-club)) calls it
with plain data and stores the resulting JSON itself. It never imports from,
writes to, or changes the behaviour of `movie_elo`.

## 1. What a member does

1. Submit a movie rec: a title and the submitting member.
2. Rank someone else's movie. Pick a bucket first: **Liked**, **Fine** or
   **Didn't like**. Then answer a short series of "Which did you like more?"
   questions against movies already in that bucket. Each answer halves the
   candidate region (binary search), so a bucket with k movies asks at most
   `ceil(log2(k + 1))` questions. There is a **Too tough** answer and a
   **Back** answer.
3. Read the results: a personal sorted list with 0.0-10.0 scores, a community
   score per movie, and a leaderboard of members.

Members cannot rank their own submissions.

## 2. Data model

```text
members    ordered list of names (strings, unique, non-empty)
movies     title -> Movie(title, submitted_by)      title is the key, exact string
lists      member -> bucket -> ordered list of titles   index 0 = best in that bucket
```

A member's full ordered list is their `liked` list, then `fine`, then
`disliked`. A liked movie always outranks a fine one for that member.
Positions are implicit (list index), so they are always a dense `0..k-1`.
A title appears at most once across a member's three lists.

Bucket constants (design choice, not a Beli fact):

| Bucket | lo | hi | baseline (single movie) |
|---|---|---|---|
| liked | 7.0 | 10.0 | 8.5 |
| fine | 4.0 | 6.9 | 5.5 |
| disliked | 0.0 | 3.9 | 2.0 |

## 3. Insertion algorithm

`Rankings.start_ranking(member, movie, bucket)` returns an immutable
`Insertion` state; `Insertion.answer(a)` returns the next state. The state
round-trips through `Insertion.to_dict()` / `Insertion.from_dict()`, so a
host can store it as JSON between stateless HTTP requests.

```text
candidates = current bucket list (k titles)
lo = 0; hi = k; history = []
while lo < hi:
    mid = (lo + hi) // 2
    ask "Which did you like more?" new vs candidates[mid]
    new        -> history.push((lo, hi)); hi = mid
    old        -> history.push((lo, hi)); lo = mid + 1
    too_tough  -> lo = hi = mid + 1        (tie: land just below the pivot, stop)
    back       -> (lo, hi) = history.pop() (no-op if history is empty)
insert new at index lo
```

- Empty bucket: zero questions.
- No comparison ever crosses a bucket.
- `finish_ranking` refuses to apply an `Insertion` whose candidate snapshot
  no longer matches the member's current bucket list.
- Remove: delete the title from its list; later entries shift up. Any pending
  search of that member should be dropped by the host, since its candidate
  snapshot is stale (`finish_ranking` would refuse it anyway).
- Re-rank: `start_rerank` validates (member, movie, bucket, is ranked) and
  searches the target bucket with the movie itself excluded from the
  candidates. The old placement stays until `finish_ranking`, which removes
  it and inserts the new one in one step. An invalid or abandoned re-rank
  changes nothing.

## 4. Score

For a bucket with `k` movies and a movie at 0-based position `p`:

```text
if k == 1: baseline
else:      round_half_up(hi - (hi - lo) * p / (k - 1), 1 decimal)
```

Arithmetic uses `fractions.Fraction`, so `5.45` rounds to `5.5` and never to
`5.4`. The best liked movie is always 10.0 and the worst liked is 7.0 once
the bucket has two or more movies.

Worked numbers (liked bucket):

| k | scores best to worst |
|---|---|
| 1 | 8.5 |
| 2 | 10.0, 7.0 |
| 5 | 10.0, 9.3, 8.5, 7.8, 7.0 |
| 10 | 10.0, 9.7, 9.3, 9.0, 8.7, 8.3, 8.0, 7.7, 7.3, 7.0 |

Mixed list, 6 liked / 3 fine / 1 disliked:
`10.0, 9.4, 8.8, 8.2, 7.6, 7.0, 6.9, 5.5, 4.0, 2.0`.

Scores of old movies move when a new one is inserted above them. That is
intended; the host shows the current numbers.

## 5. Community score and leaderboard

- **Community score** of a movie: round-half-up mean of every member's
  personal score for it; `None` ("Not ranked yet") when nobody ranked it.
  The count is shown beside it.
- **Leaderboard value** of a member: mean of the community scores of the
  movies they submitted that have at least one ranking; `None` when none do.
  Sorted by value desc (None last), then number of ranked submissions desc,
  then name. This measures how good a member's picks are to others, not how
  generous they are.

## 6. Persistence

The library does no file I/O on its own. `Rankings.to_dict()` /
`Rankings.from_dict()` give the plain shape below; `movie_beli.storage` is an
optional helper that adds a `"version": 1` field (`rankings_to_dict`,
`rankings_from_dict`) and reads/writes one JSON file atomically
(`save_rankings`, `load_rankings`, temp file + `os.replace`).

```json
{
  "version": 1,
  "members": ["Alex", "Bea"],
  "movies": [{"title": "Heat", "submitted_by": "Alex"}],
  "lists": {"Bea": {"liked": ["Heat"], "fine": [], "disliked": []}}
}
```

Any other `version` or a wrong shape (non-list members, non-object lists) is
rejected with a `ValidationError`. Names and titles are stripped, must be
1-200 characters, and must not contain control characters. The library does
not escape for HTML or URLs; the host does.

## 7. Integration

Every public call takes plain data (strings, dicts, lists) and returns plain
data or a small object with `to_dict()`. Nothing is kept between calls: the
host loads the ranking JSON, builds a `Rankings`, calls one method, and stores
`to_dict()` back. A bridge script (for example reel-club's stdin/stdout
bridge) needs only these calls:

| Step | Call | Returns |
|---|---|---|
| Load / save | `Rankings.from_dict(data)`, `r.to_dict()` | `Rankings` / dict |
| Add member | `r.add_member(name)` | `None` |
| Submit movie | `r.submit_movie(title, submitted_by)` | `Movie` (`.to_dict()`) |
| What can be ranked | `r.rankable(member)` | tuple of `Movie` |
| Start | `r.start_ranking(member, title, bucket)` or `r.start_rerank(...)` | `Insertion` |
| Store / resume | `state.to_dict()`, `Insertion.from_dict(d)` | dict / `Insertion` |
| Next question | `state.done`, `state.pivot`, `state.questions_left` | bool, title, int |
| Answer | `state.answer("new" \| "old" \| "too_tough" \| "back")` | new `Insertion` |
| Place | `r.finish_ranking(state)` (only when `state.done`) | 0-based index in bucket |
| Remove | `r.remove(member, title)` | `None` |
| Scores | `r.member_list(member)` -> `Ranked` rows; `r.score_of(member, title)`; `r.community_score(title)`; `r.movie_scores(title)` | rows have `.to_dict()`; floats or `None`; `(member, bucket, score)` tuples |
| Leaderboard | `r.leaderboard()` | `LeaderboardRow` rows (`.to_dict()`) |
| One-shot | `r.rank_movie(member, title, bucket, ask)` / `r.rerank(...)` | index; `ask(new, old)` returns an answer |

Rank step loop for one HTTP request: `state = Insertion.from_dict(stored)`;
if `state.done`, call `finish_ranking(state)`, save `r.to_dict()`, delete the
stored state; else show `state.movie` vs `state.pivot`, and on the answer
store `state.answer(a).to_dict()`. Store one pending state per member and
drop it when that member removes or re-ranks a movie.

Errors are subclasses of `MovieBeliError`: `ValidationError`,
`UnknownMemberError`, `UnknownMovieError`, `DuplicateMovieError`,
`OwnSubmissionError`. A host should map them to a 400-class response.
Constants: `BUCKETS`, `BUCKET_LABELS`, `BUCKET_RANGE`, `ANSWERS`.

## 8. Decisions

| Decision | Choice | Why |
|---|---|---|
| Movie key | exact title string, no year | matches `movie_elo`, which also keys movies by title; the submit form is title + submitter only |
| Bucket boundaries | 7/4 split, non-overlapping | keeps "every liked > every fine" true in the numbers too |
| Single movie in a bucket | fixed baseline | a lone movie should not show 10.0 |
| Too tough | land just below the pivot, stop | deterministic, zero extra questions, fixable by re-rank |
| Skip | not offered | too tough covers it in v1 |
| Leaderboard | mean score received, count tiebreak | measures pick quality, not generosity |
| Self-ranking | rejected at the API | otherwise the leaderboard is gameable |
| UI | none; library only | the hosted page lives in reel-club (Node + SQLite), which stores the JSON and calls this library through a bridge |
| Pending searches | held by the host as `Insertion.to_dict()` | nothing half-done is ever written into the ranking data |
| Members | own list in the ranking data | keeps the two systems independent |
| Not built | friend averages, want-to-watch list, member deletion, genre scoping | not needed for v1 |

## 9. Known limitations

- **Movies have no id or year.** Two spellings of one film are two movies,
  as in `movie_elo`. No `ranked_at` timestamp is stored.
- **Pending state is the host's job.** A stored `Insertion` whose candidate
  snapshot no longer matches the bucket is refused by `finish_ranking`; the
  host should drop a member's pending state when that member removes or
  re-ranks a movie.
- **Single writer.** The library has no locking; the host serialises
  changes to one ranking.
- **Not built:** member deletion, want-to-watch list, friend averages, a
  separate skip button, genre scoping, a minimum sample for the leaderboard.
