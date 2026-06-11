# Counterfactually Fixing PostgreSQL Essential Background Task Naming: From Vacuum to Housekeeping

*This document proposes an alternative mapping vocabulary for a crucial subset of PostgreSQL's configuration and diagnostic terms, for explanatory purposes. The central term `VACUUM` — a single name covering several unrelated jobs, inherited from last-century designs — actively misleads about the task's contemporary scope. The aim is to give users a coherent lens to read the real terms through, not to suggest they be changed in software.*


*Databases are commonly explained as libraries — the index is a card catalog, finding a book is a query. That metaphor covers lookups and stops. It has no room for dead rows accumulating on shelves, background maintenance, bloat, or registration wraparound, because a library's defining property is order, and the thing we need to explain is managed disorder. A campus contains a library, but it also contains buildings that get dirty, corridors that get congested, maintenance staff, a registration office, and a housing authority. The operational reality of a database fits inside a campus, not a library.*

---

## The campus and its parts

A campus (`cluster`) contains one or more districts (`databases`). Each district contains buildings (`tables`). Each building is divided into rooms (`8KB heap pages`). Each room holds tenants (`tuples` — row versions). Tenants' belongings are their data: the column values, the payload. The tenant is the metadata envelope: the registration number that created them, the one that ended their residency if they've departed, and the flags that say whether they've been stamped permanent.

Each building has a records room with filing cabinets (`indexes`) that track where tenants are. The front desk (`query planner`) consults the filing cabinets and the building's lobby status board to decide the fastest way to answer a visitor's question, then sends a runner (`executor`) to carry that route out. The filing cabinets know which rooms tenants are in; the status board knows which rooms are clean. Deciding the route and walking it are separate steps.

Each building also has an attached warehouse (`TOAST table`) for tenants with oversized belongings. Those tenants live in a normal room but keep a claim ticket (`TOAST pointer`) pointing to the warehouse. The warehouse needs its own housekeeping too, normally carried out alongside the building's.

---

## What housekeeping does

A housekeeper (`VACUUM`) visits a building and walks room by room. In each room, they do whatever combination of three jobs the room needs.

### Room turnover

When tenants leave campus (`DELETE`) or get assigned to new rooms (`UPDATE` — which in this system creates new belongings for a tenant in another room), their belongings remain in the old room. The housekeeper clears out the old things and updates the building's free-space registry (`free space map`) so the front desk knows the room has space for new arrivals.

If this work falls behind, tenants' stuff left in rooms they departed from accumulates. New arrivals get directed to fresh rooms because existing rooms aren't marked as having space. The building grows larger than its active population justifies, which is known as bloat.

Two things keep bloat sticky even when the housekeeper keeps pace with departures. The first is that emptying a room frees its space for reuse but doesn't shrink the building. The housekeeper consolidates the free space in a cleared room and tells the front desk the room is available, but the building keeps its footprint: an empty room in the middle of the building stays part of the building, and only when the rooms at the very end are all empty can the outer wall be pulled in. Reclaiming the building's actual footprint — not just freeing rooms for reuse — takes a rebuild (below).

The second is that placement is per room. A new arrival goes into a room only if that room has enough free space for their setup, and setups vary in size. The space freed inside any one room is contiguous, but it's scattered across many rooms in small amounts, so an arrival who needs more than any single room currently offers gets a fresh room opened for them — even though the building-wide total would easily have covered it. On this alone a building can keep growing, even with diligent turnover.

The housekeeper also pulls stale cards from the filing cabinets — entries pointing to rooms where the tenant they reference has already departed. The filing cabinet's physical structure (drawer splits, half-empty folders) doesn't get compacted by this; compacting the cabinet itself is a separate job, one the special crews handle (below).

### Registration stamping

Every tenant arrives with a temporary registration number drawn from a campus-wide pool (`transaction ID`). The pool is finite — about four billion numbers on a circular counter (`32-bit XID space`). The numbers go up. After four billion, they wrap to zero. The system determines whether a registration is old or new by measuring the distance around the circle: anything less than two billion ahead is "in the future," anything behind is "in the past."

A housekeeper stamps long-resident tenants as permanent (`freezing`): the housekeeper replaces the tenant's temporary registration number with a marker that means "old, always in the past, don't compare." That tenant is out of the numbering game. The stamp doesn't change anything about the tenant's belongings, their room, or who can visit them — it only changes the registration card, though writing and filing that card is itself work.

By default, a tenant must be at least fifty million registrations old (`vacuum_freeze_min_age`) before the housekeeper bothers — about 2.5% of the two-billion span before a registration would start reading as future-dated, so stamping begins long before that limit.

Every building tracks its oldest unstamped registration number (`relfrozenxid`). Every district tracks its worst building. The campus tracks its worst district. One stuck building holds back the whole campus. If the campus-wide pool approaches exhaustion, the registration office stops issuing new numbers — no new arrivals, no room changes, no move-outs. Visitors can still walk through the gates and ask questions. The campus goes read-only until someone advances the stuck building's number.

This is the wraparound emergency. The registration office shuts down because the bookkeeping system that tracks who's current and who's departed is running out of space.

### Status board updates

Every building has a lobby status board (`visibility map`) with one entry per room. Each entry has two flags:

The first flag — settled (`all-visible`) — means all tenants in this room are accounted for, no departed tenants' belongings need clearing. The second flag — completed (`all-frozen`) — means all tenants in this room have been stamped permanent.

The status board has two customers:

The next housekeeper who visits can skip rooms marked settled. Nothing to do there. This makes subsequent visits faster: instead of walking every room, they check the board and visit only the rooms that need attention.

The front desk uses the settled flag for a different purpose entirely. When a visitor asks about a tenant, the front desk consults the filing cabinet and gets a room number. Normally it would send a runner up to the room to confirm the tenant is actually there — because the filing cabinet doesn't know whether a tenant has departed since the cabinet was last updated. But if the status board says that room is settled, every tenant in it is confirmed present, and the front desk can answer from the filing cabinet alone, without sending anyone upstairs. This is the difference between consulting the filing cabinet *and* the room, versus consulting the filing cabinet *only* (`index-only scan`).

If the housekeeper falls behind and rooms lose their settled flag — any change in a room clears both flags — then the front desk loses its shortcut. The filing cabinet still works, but now a runner has to go up to every room to verify. Performance drops without the building's structure or population changing — the status board just went stale.

The completed flag serves the stamping operation. When the housekeeper is dispatched in aggressive mode (below), they need to visit every room that might still contain unstamped tenants. Rooms marked completed can be skipped — everything in them is already permanent. On a large building where most rooms haven't changed, this turns a full-building pass from hours to minutes. But if the completed flags aren't being maintained, the aggressive pass visits every room regardless, and a mostly-quiet building triggers a disproportionate amount of work.

---

## The census

The tenant census (`ANALYZE`) is a separate operation from the housekeeper's three jobs. It's a sample of the building's population: how many tenants, what's the distribution of their disciplines and demographics, how clustered they are by floor. The front desk uses this to decide which route to take when directing visitors.

A visitor asks: "Where can I find Armenian linguists?" The front desk checks the census. If the census says Armenian linguists are clustered on floors three through five, the front desk sends the visitor there directly. If the census is stale — the Armenian linguists graduated last semester and the floors are now full of Malawian physicists — the front desk sends the visitor on a wasted trip.

The census is conducted by a separate census taker (`ANALYZE` command). The census taker visits the building, samples a number of rooms, records the distributions, and updates the front desk's reference materials (`pg_statistic`).

---

## Head of housekeeping

The head of housekeeping (`autovacuum launcher`) runs continuously. They manage a pool of housekeepers — by default three (`autovacuum_max_workers`) — and dispatch them to buildings that need attention.

They maintain their own status board (`pg_stat_user_tables`, autovacuum's internal work queue), tracking across every district and every building: how many departed tenants have accumulated, how old the oldest unstamped registration is, when each building was last visited. When a housekeeper becomes available, the head sends them into the district that needs attention most — by accumulated pressure, and above all by how close any building there is to its registration deadline. Once inside, the housekeeper works through every building in that district that needs attention.

The pool size is configurable. Adding more housekeepers helps until the campus's shared infrastructure — loading docks, supply lines, corridor capacity (`shared I/O bandwidth, CPU, buffer pool`) — becomes the bottleneck. The housekeepers share a collective work-rate budget (`autovacuum_vacuum_cost_limit`). Adding housekeepers without raising the budget means each one works proportionally slower. Doubling the workers without doubling the budget gives you twice as many housekeepers each working at half speed: the same total throughput spread thinner.

---

## Housekeeper modes

### Routine

The standard dispatch. The housekeeper visits a building, walks room by room, does whatever combination of cleaning, stamping, and board-updating each room needs. They throttle themselves — working a bit, pausing, working a bit, pausing (`cost-based delay`) — so they don't disrupt the residents. If they're interrupted by other activity in the building, they give up gracefully and the building goes back into the dispatch queue.

### Aggressive

Same housekeeper, different orders. When a building's oldest unstamped registration has aged past a threshold (`autovacuum_freeze_max_age`), the head dispatches them specifically to advance the building's registration horizon, and they work differently. They ignore the settled flag: a room can be settled — no belongings left to clear — and still hold tenants who were never stamped, so they have to go in and check it. What they can still skip are rooms marked completed, where every tenant is already permanent. For every unstamped tenant they find, they write a fresh permanent card — real work and real paperwork, which is why a freeze pass is I/O-heavy.

On a well-maintained building where the completed flags are current, most rooms are skipped and the pass is cheap. On a building where the flags have gone stale, almost every room needs an actual visit and the pass is expensive.

### Manual

The campus owner (`DBA`) can dispatch a housekeeper directly, bypassing the head's normal scheduling. Same housekeeper, same skills, different chain of command. Used for maintenance windows, investigation, or catching up after a deferred period.

---

## The special crews

There are two ways to rebuild a bloated building. Both end with a compact building; what differs is whether they close it to get there. That axis, locking versus online, decides whether a rebuild is something you schedule and announce or something you run against live traffic. It is also the axis the tool names hide — from PostgreSQL 19, inside a single command.

### The closed-building crew (locking)

If a building has accumulated severe bloat — empty rooms, scattered tenants, far more space than the resident count justifies — routine housekeeping can't fix it. The housekeeper reclaims space within rooms but can't compact the building; for that the structure has to come down and go back up compact.

The closed-building crew takes an exclusive lock. Nobody gets in or out. It builds a new compact building with the same tenants, fresh filing cabinets, a clean status board, and reopens at the correct size. It removes the bloat completely, in exchange for a full lockout while it runs. In PostgreSQL this is `VACUUM FULL` and its near-twin `CLUSTER` (the same rewrite, tenants written in a chosen order). From PostgreSQL 19 an in-core command, `REPACK`, folds the two under one name (the old spellings remain). Bare `REPACK` locks exactly as they do — and that is the trap, because for about fifteen years "repack" has meant the *online* extension. The word now names a command whose *default* is the closed-building rewrite, with the open-building mode behind an option (next section). Someone who reaches for `REPACK` expecting `pg_repack`'s manners shuts the building.

### The open-building crew (online)

A different crew builds the new building on the adjacent lot while the old one stays open. A relay intercepts every change in the old building and replicates it to the new one in real time; the new building gets its own filing cabinets and status board from scratch. When it has caught up, the crew swaps the address plates and the old building comes down. For tenants and visitors the only disruption is a brief lockout at the instant of the swap.

In PostgreSQL 18 and earlier this is `pg_repack` or `pg_squeeze` — extensions, not core; the online rebuild of a *table* lives outside core. From 19 the crew is in core: `REPACK CONCURRENTLY`, built on `pg_squeeze`'s method — logical decoding as the relay — by Antonín Houska, `pg_squeeze`'s author.

The price of staying open: two buildings run at once, so space roughly doubles. The swap needs a moment of exclusive access; if a visitor at the front desk won't leave, it waits, and if it waits too long the crew gives up and retries. A risk is to have the crew killed mid-operation — power loss, crash, abort: the original building survives and no tenants are harmed, but the relay would still be wired into its front desk adding overhead to every change, and the half-built replacement sitting on the lot consuming space until cleared. And the relay has a standing cost the closed crew doesn't — the variants that capture changes through logical decoding (`pg_squeeze`, and from 19 `REPACK CONCURRENTLY`) hold a replication slot for the whole rebuild, which pins the campus registration horizon for as long as the job runs. The first in-core release carries its own sharp edges, stated in its own commit notes: only one `REPACK CONCURRENTLY` may run campus-wide at a time, the final swap can lose a deadlock against traffic and abort, and a crew killed mid-job can orphan its slot — the horizon stays pinned until someone notices and clears it by hand.

### The filing cabinets got there first

One part of "rebuild" was solved online, in core, years before the other: the cabinets themselves. `REINDEX CONCURRENTLY` builds a fresh cabinet beside the old one and swaps it in under a lock that blocks neither reads nor writes — the machinery of `CREATE INDEX CONCURRENTLY` — then drops the bloated original. Compacting a *cabinet* has needed no extension and no window since PostgreSQL 12; rebuilding the *building* needs one or the other through 18. Keep the two apart, because "bloat" collapses them and their remedies arrived years apart: the cabinets got their in-core online command first, and the building only catches up in 19.

---

## The compliance officer

When a building approaches its registration deadline and routine stamping clearly won't catch up, the head of housekeeping calls in the compliance officer (`failsafe vacuum`, PostgreSQL 14). They're a stern bureaucrat from the campus housing authority whose only job is processing registrations before deadlines.

They don't care about cleaning, don't care about the census, don't update the status board. They visit every room with unstamped tenants, stamp them permanent at maximum speed — no throttling, no pausing for residents — and leave when the deadline is no longer in immediate danger.

The compliance officer arriving means the situation got away from routine maintenance. That's not necessarily the housekeeper's fault — the building might have an unusually high arrival rate, or something blocked routine work long enough for pressure to accumulate past safe levels.

---

## When things go wrong

### Freshman season

Thousands of new tenants arriving. Corridors packed. The housekeeper is present but throttled — they work slowly on purpose so they don't make the congestion worse. The building accumulates departed tenants' belongings faster than the housekeeper can clear them. Bloat grows. If it persists, the housekeeper never catches up and the building needs a rebuild crew.

Fix: raise the housekeeper's work-rate limit (`autovacuum_vacuum_cost_limit`) or add more housekeepers. Both have the same constraint — the campus's shared infrastructure sets a ceiling on total housekeeping throughput.

### The researcher's deadline

One researcher is on a publication deadline, running around the building consulting colleagues. Nothing in the building can be cleared until their work is finished, because they might need to visit any room at any time. The housekeeper can still stamp tenants permanent — that doesn't move or change anything they depend on — but they can't clear departed tenants' belongings.

The stuck belongings contain old registration numbers. The building's registration horizon (`relfrozenxid`) can't advance past them. The stamping work proceeds but the building-level deadline doesn't move. One building's stuck deadline holds back the district. The district holds back the campus.

Fix: find the researcher and get them to finish — or cancel their work. In PostgreSQL terms: identify the long-running transaction (`pg_stat_activity`), the unused replication slot (`pg_replication_slots`), or the abandoned prepared transaction (`pg_prepared_xacts`) that's pinning the horizon, and close it.

### Head of housekeeping sent home

The campus owner has dismissed the head (`autovacuum = off`). No dispatching happens. The housekeeper pool sits idle. Pressure accumulates on all jobs silently across every building. Usually a configuration mistake. Note: even with the head dismissed, the campus will still trigger emergency housekeeping if a building approaches the wraparound deadline.

### Pool too small

The head is dispatching correctly but the pool is too small or the work-rate budget is too low. Buildings queue up waiting for a housekeeper. Some buildings go a long time between visits. The head's status board shows the backlog but can't resolve it without more resources.

### Status board doesn't reflect actual pressure

Historically, the head's dispatch board tracked departed tenants as the signal for cleaning urgency. Buildings with very few departures — high arrival rate, not much turnover — looked idle on the board even when their stamping pressure was mounting. A building full of new arrivals who all needed registration stamping produced no departed tenants, which meant the head never dispatched a housekeeper, which meant registrations piled up silently.

This was fixed in PostgreSQL 13, which added arrival-rate signals to the dispatch board (`autovacuum_vacuum_insert_threshold`, `autovacuum_vacuum_insert_scale_factor`). Before 13, insert-heavy buildings were invisible to the head of housekeeping.

---

## The configuration knobs in campus terms

The knobs that control housekeeping divide into a few groups.

**How often to clean:** how many departed tenants accumulate before the head dispatches a housekeeper (`autovacuum_vacuum_threshold` + `autovacuum_vacuum_scale_factor`). A building with 10,000 tenants and a scale factor of 0.2 triggers cleaning after 2,000 departures. For large buildings, 20% is too high — cleaning is triggered too late and bloat accumulates. Reducing the scale factor or the fixed threshold makes the head more aggressive about dispatching.

**How often to census:** same structure, separate thresholds (`autovacuum_analyze_threshold` + `autovacuum_analyze_scale_factor`). The census can trigger more or less often than cleaning.

**How fast the housekeeper works:** the work-rate budget (`autovacuum_vacuum_cost_limit`) and the pause duration (`autovacuum_vacuum_cost_delay`). The housekeeper does a burst of work, pauses, does another burst. Higher budget or shorter pauses mean faster housekeeping and more load on the campus. The budget is shared across all active housekeepers.

**When to stamp:** how old a tenant must be before the housekeeper bothers stamping (`vacuum_freeze_min_age`). How old the building's oldest unstamped tenant can get before the head dispatches an aggressive pass (`autovacuum_freeze_max_age`). How close to the deadline the building gets before the compliance officer is called in (`vacuum_failsafe_age`). These are three thresholds at increasing levels of urgency — the first measured by a tenant's own age, the other two by the age of the building's oldest unstamped tenant.

**How many housekeepers:** the pool size (`autovacuum_max_workers`). More housekeepers means more buildings can be serviced simultaneously but each one works proportionally slower unless the total budget is also raised.
