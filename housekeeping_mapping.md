# Counterfactually Fixing PostgreSQL Essential Background Task Naming: From Vacuum to Housekeeping

*This document proposes an alternative mapping vocabulary for a crucial subset of PostgreSQL's configuration and diagnostic terms, for explanatory purposes. The central term `VACUUM` — a single name covering several unrelated jobs, inherited from last-century designs — actively misleads about the task's contemporary scope. The aim is to give users a coherent lens to read the real terms through, not to suggest they be changed in software.*


*Databases are commonly explained as libraries — the index is a card catalog, finding a book is a query. That metaphor covers lookups and stops. It has no room for dead rows accumulating on shelves, background maintenance, bloat, or registration wraparound, because a library's defining property is order, and the thing we need to explain is managed disorder. A campus contains a library, but it also contains buildings that get dirty, corridors that get congested, maintenance staff, a registration office, and a housing authority. The operational reality of a database fits inside a campus, not a library.*

---

## The campus and its parts

A campus (`cluster`) contains one or more districts (`databases`). Each district contains buildings (`tables`). Each building is divided into rooms (`8KB heap pages`). Each room holds tenants (`tuples` — row versions) and their belongings (the column values, the data payload).

Every room has a door plate visible from the corridor. The plate lists each tenant inside, one line per tenant, showing their check-in number, their check-out number if they've departed, and whether they've been stamped permanent. The housekeeper reads the plate from the corridor to decide what needs doing in the room.

Each building has a records room with filing cabinets (`indexes`) that track where tenants are. The front desk (`query planner`) consults the filing cabinets and the building's lobby status board to decide the fastest way to answer a visitor's question, then sends a runner (`executor`) to carry that route out. The filing cabinets know which rooms tenants are in; the status board knows which rooms are clean. Deciding the route and walking it are separate steps.

Each building also has an attached warehouse (`TOAST table`) for tenants with oversized belongings. Those tenants live in a normal room but keep a claim ticket (`TOAST pointer`) pointing to the warehouse. The warehouse needs its own housekeeping too, normally carried out alongside the building's.

---
## The ticket dispenser

There is a ticket dispenser at each district gate. Everyone entering the district — visitors asking questions and movers placing or removing tenants — pulls a ticket. The tickets are numbered sequentially from a single campus-wide counter. When a mover places a tenant in a room, they write their own ticket number on the room's door plate as the tenant's check-in number (`xmin`). When a different mover later moves a tenant out or reassigns them to another room, that mover writes their ticket number on the door plate as the tenant's check-out number (`xmax`). The tenant's belongings stay in the old room. The door plate now has two numbers for that tenant: one saying when they arrived in the sequence, one saying when they left.

When a tenant is reassigned — moved to a new room with updated belongings — the mover writes the same ticket number in two places: the check-out field on the old room's door plate, and the check-in field on the new room's door plate. One ticket, two writes. The old belongings stay in the old room. The new belongings go in the new room. Nothing is overwritten, nothing is moved.

The counter is finite. Thirty-two bits, about four billion numbers, arranged in a circle — after four billion, it wraps to zero. The system tells old from new by measuring the distance around the circle: anything less than two billion ahead of the current position is in the future, anything behind is in the past. That comparison holds as long as no number is so old it has drifted more than halfway around the ring, at which point it flips from past to future and the system can no longer tell the difference. This is why the housekeeper stamps long-resident tenants permanent: their check-in number on the door plate is replaced with a marker meaning "old, always in the past, don't compare." Out of the numbering game. Can never wrap around. Every building tracks its oldest unstamped number (`relfrozenxid`). One stuck building holds back the whole campus.

## Ticket holders

A visitor or mover holds their ticket from the moment they pull it until they leave the district and hand it back. While they do, the housekeeper has a problem.

The housekeeper cannot just walk into a room and clear a departed tenant's belongings. They have to check who is still sitting at the district food court. If a visitor is sitting there holding ticket number one thousand, and a door plate's check-out field for a departed tenant says one thousand-and-one, that visitor entered the district before the tenant left. From where that visitor sits in the sequence, the tenant was still there when they walked in. Clearing those belongings could break whatever that visitor came to find out. The housekeeper leaves the room untouched until that visitor finishes and hands back their ticket.

The oldest ticket still held by anyone at the food court — visitor or mover — is the district's horizon (`OldestXmin`). The housekeeper reads the check-out field on each door plate entry and compares it to the horizon. Below the horizon: everyone who was in the district when this tenant left has since handed back their ticket, which means the belongings can be cleared. At or above the horizon: someone might still need to see them.

One idle visitor is enough. Someone who pulled ticket number one thousand, got their answer, and sat down at the food court to read a newspaper — an application that opened a transaction and never committed — keeps the horizon pinned at one thousand while the dispenser climbs into the millions. Every room checked out after one thousand is unclearable across the entire district. Not because those rooms have anything to do with that visitor's question, but because the housekeeper cannot know which rooms the visitor might still consult.

## The auditor

Some campuses are legally required to keep a duplicate of every door plate entry filed at an offsite auditor's office — the best we can do here for now as to mapping PostgreSQL's replication onto the campus metaphor: There's just no real world sense to having an exact copy of an entire campus somewhere. Just like the library metaphor strains at dead tuple cleanup, so does the campus metaphor when replication comes into play.

The auditor receives copies in sequence and processes them in order, always somewhat behind. They leave a standing note at the main campus gate — not at any district gate, but at the central entrance: "I have reviewed up to entry number one million." That note pins the horizon campus-wide. A housekeeper in any district, in any building, cannot clear belongings whose check-out number is above the review threshold — because the auditor hasn't processed them yet, and the duplicate records must remain reconstructable until they have.

If the auditor goes silent — office closed, lost their lease, abandoned their obligations — the note stays at the gate. The campus accumulates unclearable belongings from te threshold forward, in every district, indefinitely, for a consumer who is never coming back. The note must be pulled manually before the horizon can advance.

If the campus's own records are destroyed — fire, flood, administrative catastrophe — the auditor's duplicate entries are the legally valid backup from which the campus can be reconstructed. This is why the requirement exists.

Prepared transactions (PREPARE TRANSACTION) work similarly: paperwork left at the central registrar's desk, neither confirmed nor cancelled. The registrar set aside resources and is waiting for a signature one way or the other. Until it comes, that paperwork holds its ticket number open at the main gate — not at any district gate — pinning the horizon campus-wide for all districts.


## What housekeeping does

A housekeeper (`VACUUM`) visits a building and walks room by room. In each room, they read the door plate and do whatever combination of the three jobs the room needs — the board entry re-checked after any clearing or stamping. A room never wants a census; that's building-level work (below).

### Room turnover

When tenants leave campus (`DELETE`) or get reassigned to new rooms (`UPDATE` — which creates new belongings in another room while the old belongings stay behind), the housekeeper clears out the old belongings and updates the building's free-space registry (`free space map`) so the front desk knows the room has space for new arrivals.

If this work falls behind, belongings left in rooms by departed tenants accumulate. New arrivals get directed to fresh rooms because existing rooms aren't marked as having space. The building grows larger than its active population justifies, which is known as bloat.

Two things keep bloat sticky even when the housekeeper keeps pace with departures. The first is that emptying a room frees its space for reuse but doesn't shrink the building. The housekeeper consolidates the free space in a cleared room and tells the front desk the room is available, but the building keeps its footprint: an empty room in the middle of the building stays part of the building, and only when the rooms at the very end are all empty can the outer wall be pulled in.

The second is that placement is per room. A new arrival goes into a room only if that room has enough free space for their setup, and setups vary in size. The space freed inside any one room is contiguous, but it's scattered across many rooms in small amounts, so an arrival who needs more than any single room currently offers gets a fresh room opened for them — even though the building-wide total would easily have covered it. On this alone a building can keep growing, even with diligent turnover.

The housekeeper also pulls stale entries from the filing cabinets — entries pointing to rooms where the tenant they reference has already departed.

### Registration stamping

The ticket dispenser section above explains why this is necessary: the campus-wide counter is finite and circular, and temporary registration numbers that sit on door plates too long eventually wrap around and read as future-dated. The housekeeper stamps long-resident tenants permanent (`freezing`), replacing the check-in number on their door plate entry with a marker meaning "old, always in the past, don't compare."

By default, a tenant must be at least fifty million registrations old (`vacuum_freeze_min_age`) before the housekeeper bothers — about 2.5% of the two-billion span before a registration would start reading as future-dated, so stamping begins long before that limit.

Every building tracks its oldest unstamped registration number (`relfrozenxid`). Every district tracks its worst building. The campus tracks its worst district. One stuck building holds back the whole campus. If the campus-wide pool approaches exhaustion, the registration office stops issuing new numbers — no new arrivals, no room changes, no move-outs. Visitors can still walk through the gates and ask questions. The campus goes read-only until someone advances the stuck building's number.

This is the wraparound emergency. The registration office shuts down because the bookkeeping system that tracks who's current and who's departed is running out of space.

### Status board updates

Every building has a lobby status board (`visibility map`) with one entry per room. Each entry has two flags:

The first flag — settled (`all-visible`) — means every tenant listed on that room's door plate is accounted for, no departed tenants' belongings need clearing. The second flag — completed (`all-frozen`) — means every tenant listed on that room's door plate has been stamped permanent.

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

The head only picks the building. The worker, once there, decides what it needs: a cleaning round (which stamps as it goes, and may be triggered by registration age alone), a census, or both — each decision against its own thresholds (`autovacuum_vacuum_*` for the round, `autovacuum_analyze_*` for the census). One visit, separate triggers.

---

## Head of housekeeping

The head of housekeeping (`autovacuum launcher`) runs continuously. They manage a pool of housekeepers — by default three (`autovacuum_max_workers`) — and dispatch them to buildings that need attention.

They maintain their own ledger (`pg_stat_user_tables`, autovacuum's internal work queue), tracking across every district and every building: how many departed tenants have accumulated, how old the oldest unstamped registration is, when each building was last visited. When a housekeeper becomes available, the head sends them into the district that needs attention most — by accumulated pressure, and above all by how close any building there is to its registration deadline. Once inside, the housekeeper works through every building in that district that needs attention.

The pool size is configurable. Adding more housekeepers helps until the campus's shared infrastructure — loading docks, supply lines, corridor capacity (`shared I/O bandwidth, CPU, buffer pool`) — becomes the bottleneck. The housekeepers share a collective work-rate budget (`autovacuum_vacuum_cost_limit`). Adding housekeepers without raising the budget means each one works proportionally slower. Doubling the workers without doubling the budget gives you twice as many housekeepers each working at half speed: the same total throughput spread thinner.

---

## Housekeeper modes

### Routine

The standard dispatch. The housekeeper visits a building, walks room by room, reads each door plate, does whatever combination of cleaning, stamping, and board-updating each room needs. They throttle themselves — working a bit, pausing, working a bit, pausing (`cost-based delay`) — so they don't disrupt the residents. If they're interrupted by other activity in the building, they give up gracefully and the building goes back into the dispatch queue.

### Aggressive

Same housekeeper, different orders. When a building's oldest unstamped registration has aged past a threshold (`autovacuum_freeze_max_age`), the head dispatches them specifically to advance the building's registration horizon, and they work differently. They ignore the settled flag: a room can be settled — no belongings left to clear — and still have tenants listed on its door plate who were never stamped, so they have to go in and check. What they can still skip are rooms marked completed, where every entry on the door plate is already permanent. For every unstamped tenant they find, they update the door plate entry — replacing the temporary check-in number with a permanent stamp. Real work and real paperwork, which is why a freeze pass is I/O-heavy.

On a well-maintained building where the completed flags are current, most rooms are skipped and the pass is cheap. On a building where the flags have gone stale, almost every room needs an actual visit and the pass is expensive.

### Manual

The campus administrator can dispatch a housekeeper directly, bypassing the head's normal scheduling. Same housekeeper, same skills, different chain of command. Used for maintenance windows, investigation, or catching up after a deferred period.

### Special: Out of Scope

There is an additional command also having vacuum in its name, `VACUUM FULL`, which isn't run by the autovacuum daemon nor does it perform any incremental dead row removal, visibility map updates or tuple freezing. It does something else entirely: rewriting a whole table from scratch which then implicitly results in zero dead rows, all frozen tuples and fresh compact indexes. Version 19 adds `REPACK` which subsumes `VACUUM FULL` while expanding its functionality, leveraging established terminology from the popular `pg_repack` extension.

---

## The compliance officer

When a building approaches its registration deadline and routine stamping clearly won't catch up, the head of housekeeping calls in the compliance officer (`failsafe vacuum`, PostgreSQL 14). They're a stern bureaucrat from the campus housing authority whose only job is processing registrations before deadlines.

They don't care about cleaning, don't care about the census, don't update the status board. They visit every room with unstamped tenants on the door plate, stamp them permanent at maximum speed — no throttling, no pausing for residents — and leave when the deadline is no longer in immediate danger.

The compliance officer arriving means the situation got away from routine maintenance. That's not necessarily the housekeeper's fault — the building might have an unusually high arrival rate, or something blocked routine work long enough for pressure to accumulate past safe levels.

---

## When things go wrong

### Freshman season

Thousands of new tenants arriving. Corridors packed. The housekeeper is present but throttled — they work slowly on purpose so they don't make the congestion worse. The building accumulates departed tenants' belongings faster than the housekeeper can clear them. Bloat grows.

Fix: raise the housekeeper's work-rate limit (`autovacuum_vacuum_cost_limit`) or add more housekeepers. Both have the same constraint — the campus's shared infrastructure sets a ceiling on total housekeeping throughput.

### The researcher's deadline

One researcher is on a publication deadline, running around the building consulting colleagues. Nothing in the building can be cleared until their work is finished, because they might need to visit any room at any time. The housekeeper can still stamp tenants permanent — that doesn't move or change anything they depend on — but they can't clear departed tenants' belongings.

The departed tenants' door plate entries carry old registration numbers. The building's registration horizon (`relfrozenxid`) can't advance past them. The stamping work proceeds but the building-level deadline doesn't move. One building's stuck deadline holds back the district. The district holds back the campus.

Fix: find the researcher and get them to finish — or cancel their work. In PostgreSQL terms: identify the long-running transaction (`pg_stat_activity`), the unused replication slot (`pg_replication_slots`), or the abandoned prepared transaction (`pg_prepared_xacts`) that's pinning the horizon, and close it.

### Head of housekeeping sent home

The campus administrator has dismissed the head (`autovacuum = off`). No dispatching happens. The housekeeper pool sits idle. Pressure accumulates on all jobs silently across every building. Usually a configuration mistake. Note: even with the head dismissed, the campus will still trigger emergency housekeeping if a building approaches the wraparound deadline.

### Pool too small

The head is dispatching correctly but the pool is too small or the work-rate budget is too low. Buildings queue up waiting for a housekeeper. Some buildings go a long time between visits. The head's ledger shows the backlog but can't resolve it without more resources.

### Ledger doesn't reflect actual pressure

Historically, the head's ledger tracked departed tenants as the signal for cleaning urgency. Buildings with very few departures — high arrival rate, not much turnover — looked idle in the ledger even when their stamping pressure was mounting. A building full of new arrivals who all needed registration stamping produced no departed tenants, which meant the head never dispatched a housekeeper, which meant registrations piled up silently.

This was fixed in PostgreSQL 13, which taught the head to count new arrivals (`n_ins_since_vacuum` in `pg_stat_all_tables`), not just departures. The thresholds for when arrivals alone trigger a dispatch are `autovacuum_vacuum_insert_threshold` and `autovacuum_vacuum_insert_scale_factor`.

---

## The configuration knobs in campus terms

The knobs that control housekeeping live in the campus standing orders (postgresql.conf), most of them overridable for a single building (per-table storage parameters), and divide into a few groups.

**How often to clean:** how many departed tenants accumulate before the head dispatches a housekeeper (`autovacuum_vacuum_threshold` + `autovacuum_vacuum_scale_factor`). A building with 10,000 tenants and a scale factor of 0.2 triggers cleaning after 2,000 departures. For large buildings, 20% is too high — cleaning is triggered too late and bloat accumulates. Reducing the scale factor or the fixed threshold makes the head more aggressive about dispatching.

**How often to census:** same structure, separate thresholds (`autovacuum_analyze_threshold` + `autovacuum_analyze_scale_factor`). The census can trigger more or less often than cleaning.

**How fast the housekeeper works:** the work-rate budget (`autovacuum_vacuum_cost_limit`) and the pause duration (`autovacuum_vacuum_cost_delay`). The housekeeper does a burst of work, pauses, does another burst. Higher budget or shorter pauses mean faster housekeeping and more load on the campus. The budget is shared across all active housekeepers.

**When to stamp:** how old a tenant must be before the housekeeper bothers stamping (`vacuum_freeze_min_age`). How old the building's oldest unstamped tenant can get before the head dispatches an aggressive pass (`autovacuum_freeze_max_age`). How close to the deadline the building gets before the compliance officer is called in (`vacuum_failsafe_age`). These are three thresholds at increasing levels of urgency — the first measured by a tenant's own age, the other two by the age of the building's oldest unstamped tenant.

**How many housekeepers:** the pool size (`autovacuum_max_workers`). More housekeepers means more buildings can be serviced simultaneously but each one works proportionally slower unless the total budget is also raised.
