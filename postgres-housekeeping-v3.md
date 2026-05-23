# How housekeeping actually works in PostgreSQL

PostgreSQL's VACUUM command does four things. It clears out dead rows, updates planner statistics, maintains the visibility map, and freezes old transaction IDs to prevent wraparound. These four jobs are operationally distinct — different triggers, different costs, different failure modes — but they share an implementation: one heap scan that does whatever combination of tasks each page needs. The bundling is efficient. The name is misleading. "Vacuum" suggests cleanup; three of the four functions aren't cleanup at all.

The name made sense once. Stonebraker's POSTGRES project at Berkeley, starting in 1986, had an unusual property: it never overwrote data. An UPDATE didn't modify a row in place — it wrote a new version and left the old one sitting there. A DELETE didn't remove anything — it just marked a version as no longer current. The point of all this was time-travel queries. A user could ask for the state of a table as it existed at any past moment, and the system could answer, because every version it had ever produced was still physically present on disk. A background daemon called the "vacuum cleaner" swept the oldest versions off expensive primary storage onto cheaper archival media. The name described the job precisely: it cleaned up what time-travel no longer needed.

The time-travel query interface was dropped in 1995, when POSTGRES became Postgres95 and then PostgreSQL. The storage architecture was not. It is still running. PostgreSQL's MVCC — the mechanism that lets concurrent transactions each see a consistent snapshot of the database — is the same no-overwrite, version-accumulating heap. Old tuple versions remain physically in place until something comes along and determines that no active transaction can see them anymore. That something is still called the vacuum cleaner.

But its job description kept changing. Once time-travel was gone, the daemon stopped sweeping versions to archive and started simply removing dead ones. Then it picked up statistics collection, because it was already visiting every page and the planner needed distribution data. Then visibility map maintenance, because the information it gathered could save future passes from revisiting clean pages. Then transaction ID freezing, because the system's finite ID space needed periodic renewal and the daemon was already doing the heap scan. Four jobs accreted over four decades, under a name inherited from the first.

Phil Karlton said there are only two hard problems in computer science: cache invalidation and naming things. Most of what makes a database hard is the first — data lives across CPU caches, main memory, disk controller buffers, persistent storage, and eventually archive, and at every boundary the system has to decide what's live, what's stale, and what needs refreshing. VACUUM is a pure instance of the second: a single command that does four distinct things, named after one of them.

Imagine someone today could use the very time-travel feature they were removing — use it literally — to reach back to 1995 and tell the developers what the next thirty years would pile onto their vacuum command. What follows is the result: the operation renamed with hindsight, the four functions given distinct names under a single metaphor: housekeeping.

---

## The counterfactual: building maintenance

A PostgreSQL cluster is a campus. The campus is organized into districts (databases), each containing many buildings (tables). Each building has rooms (8KB heap pages) where tenants (tuples) live. Each building also has a records room with filing cabinets (indexes) that track where tenants are, but the rooms themselves are the primary unit of building life.

Most tenants' belongings fit comfortably in their assigned room. Some tenants have oversized possessions that don't fit — long manuscripts, bulky collections. Every building has an attached warehouse for these; the tenant lives in a room as usual and keeps a small claim ticket pointing to their items in the warehouse. The warehouse needs its own housekeeping on the same schedule as the building, but it's largely out of sight for everyday building operations.

The campus has one head of housekeeping. She manages a pool of housekeepers — by default three, configurable — who are dispatched on demand to whichever buildings most need attention. A housekeeper is not assigned to any one building; she visits the buildings the head sends her to, does whatever work they need, and returns to the pool to be reassigned. At any moment, up to three housekeepers can be working simultaneously, each in a different building, possibly in different districts.

When a housekeeper visits a building, she does four jobs.

### Room turnover

When tenants move out — voluntarily (DELETE) or because they got upgraded to a new room (UPDATE, which in PostgreSQL leaves the old version behind as a *departed tenant*) — they leave their belongings in the old room. The housekeeper clears out the belongings and updates the building's registry: this part of the room is now free space available for future arrivals.

If routine cleanup falls behind, departed tenants' belongings accumulate in rooms across the building. New arrivals get directed to fresh rooms because existing rooms aren't yet cleared. Eventually cleanup catches up, but by then the new arrivals have already settled in the fresh rooms. The building has grown larger than its active population justifies — old rooms with cleared space, scattered active tenants, and walks between active rooms that are longer than they should be. This is *bloat*.

The natural question: why doesn't the front desk just place subsequent new arrivals in the cleared old rooms, so the building eventually compacts itself? It does try. The front desk consults a registry — maintained by the housekeeper — tracking the largest contiguous free zone in each room. New arrivals are directed to existing rooms with enough space whenever possible.

The reason this doesn't fully prevent bloat is fragmentation. The tenants are researchers with different setup requirements. One needs a corner of a sofa and a laptop. Another needs a workstation, a rack of equipment, and clear floor space around it. Another needs a desk, a whiteboard, and a bookshelf along an entire wall. After several researchers have moved out, the cleared zones in a room have specific sizes — the equipment space here, the laptop corner there. A new arrival needing a desk-plus-whiteboard-plus-bookshelf configuration can fit only into a contiguous space large enough; if no existing room has such a zone, they get directed to a fresh room. The smaller cleared zones go on accumulating across the building, individually too small for some new arrivals but adding up to substantial unused space.

The registry also tracks only the *largest* contiguous zone per room, not the sum of available space. A room with one large cleared zone and a room with many small zones totaling the same area look equally accommodating to the front desk if the largest zone matches what the arrival needs. But the latter room has dead space hiding around the smaller researchers, space that can't be used because it's the wrong size for typical new arrivals.

Over time, the building grows because new arrivals with larger configurations keep being directed to fresh rooms, while older rooms accumulate fragmented unused zones that can't host them. The housekeeper can keep each room clean and the registry accurate, but she has no mechanism for rearranging existing tenants to consolidate their setups. The rebuild crews do — that's part of what they're for.

### Tenant census

The building has a front desk. Guests arrive and ask for directions: "where can I find a Spanish-speaking architect," "how many mature students are in residence," "is there anyone with the surname Nguyen on floors 3 through 7." The front desk wants to answer these efficiently — choosing the best route to the right tenants — without sending someone to knock on every door.

To make that possible, the housekeeper periodically takes a census of the resident population. Not a complete count — a sample, enough to estimate distributions. How many tenants in total. What's the distribution of their disciplines and demographics. How clustered they are by floor. The front desk uses this census to choose routes with reasonable accuracy.

The census goes stale as the population changes. New arrivals shift the distribution. Departures shift it the other way. If the census hasn't been refreshed recently, the front desk's routing gets less reliable, which means guests get sent the long way around or to floors where the tenants they need have already left.

### Permanent registration stamping

Every tenant, when they arrive at a building, is issued a temporary registration number drawn from a campus-wide pool. The pool is finite. New arrivals across the entire campus consume numbers from the same pool.

The constraint is managed by recycling. After a tenant has been in residence long enough to be considered permanent, the housekeeper visits their room, confirms they're settled, and stamps their registration as permanent. Once stamped permanent, the tenant's registration number is no longer subject to recycling — they're durable, and the records office can free up the number's slot for future use elsewhere.

Each building has its own internal deadline based on how old its oldest unstamped tenant is. When a building approaches its deadline, an emergency response gets triggered for that specific building. If multiple buildings fall behind simultaneously and the campus-wide pool of registration numbers gets too close to exhaustion, the housing authority shuts down new arrivals across the entire campus until emergency procedures restore capacity. This is the *wraparound emergency*.

The stamping pressure is genuinely separate from the room-turnover pressure. A building can have very few departures (insert-heavy workload) and still accumulate stamping pressure because new arrivals keep entering the system. A building can have lots of departures (update-heavy workload) without much stamping pressure if the arrival rate is low. The two pressures share a housekeeper but they don't share causes.

### Status board updates

In the building's lobby, the housekeeper maintains a status board with one entry per room. Each entry has two flags:

- **Settled**: all current tenants in this room are properly placed, no one is in the middle of moving in or out, no departed-tenant belongings need clearing
- **Completed**: all current tenants in this room have been stamped permanent, not just settled

The housekeeper updates the board as a side effect of her other work. When she visits a room and finds it clean of departed belongings, she sets the settled flag. When she visits a room and stamps all the remaining tenants permanent, she sets the completed flag. When any change happens in a room — new arrival, departure, anything — both flags get cleared. A single write to a room that was fully completed undoes both flags, which means the next housekeeper and the next aggressive-mode pass both have to revisit it.

The board has two purposes. First, the next housekeeper who visits the building uses it: she can skip rooms that are still marked settled, because there's nothing to do there. Second, the front desk uses it for guest questions: if a guest asks about a tenant, and the records cabinets indicate the tenant is in room 304, and the board shows room 304 as settled, the front desk can answer the guest directly without sending anyone to the room. The combination of records-cabinet entries plus the settled flag gives the front desk a faster path to answers.

### The housekeeper does all four jobs in one visit

The housekeeper visits a room and performs whatever combination of the four tasks is appropriate based on what she finds. She doesn't have separate trips for separate tasks. The bundling is operationally efficient because the cost of getting to each room dominates the cost of the work done in the room.

The bundling is also pedagogically confusing because the four tasks are unrelated. Room turnover is about reclaiming space. Tenant census is about helping the front desk. Stamping is about avoiding the registration deadline. Status board updates are about making subsequent visits faster. They share a housekeeper, not a purpose.

---

## The staff and their modes

### The head of housekeeping

The head of housekeeping has a status board of her own — campus-wide, distinct from the per-building status boards in each lobby. She tracks across every district and every building: how many departed tenants in each building, how many tenants approaching the stamping deadline, when each building was last visited, how each visit went.

Her job is dispatching the housekeeper pool. When a housekeeper becomes available (either because she just finished a building or because the head is starting a new round), the head looks at her board and sends the housekeeper to whichever building most needs attention. Buildings with more pressure get visited more often. Buildings with no pressing work might go a long time between visits.

The pool size (default three) is configurable. Adding more housekeepers makes catch-up faster across the campus until the campus's shared infrastructure (loading docks, supply lines, parking) becomes the bottleneck. Beyond that point, more housekeepers actually slow the work down because they spend more time waiting for shared resources than working.

### Routine housekeepers

The standard mode. When dispatched to a building, the housekeeper does all four tasks per room visited, throttles herself based on configurable cost limits so the residents aren't constantly inconvenienced, and can be interrupted by other operations in the building. If she gets interrupted, she gives up gracefully and tries again later. The building goes back into the head's dispatch queue.

### Aggressive housekeepers

Same person, different orders. When the stamping pressure on a building has accumulated past a threshold, the head of housekeeping dispatches the housekeeper with instructions to visit every room, including rooms marked completed on the lobby board. The aggressive mode ignores the board's "this room is fine" hints and revisits everywhere, in order to advance the building's overall stamping horizon. This costs more in the short term but is necessary periodically.

### The owner's direct hire

Sometimes the campus owner wants specific work done outside the head of housekeeping's normal schedule — preparing for a maintenance window, investigating something, catching up after a deferred period. The owner can dispatch a housekeeper directly, bypassing the head's dispatcher. Same housekeeper, same skills, different chain of command.

### The demolition and reconstruction crew

If a building has accumulated severe bloat — empty floors, scattered tenants, much more space than the resident count justifies — routine housekeeping can't fix it. Room turnover reclaims space *within* the existing room layout but doesn't compact the building back to its proper size. For that, you need to close the building, demolish the structure, and rebuild it compact.

The scheduled-closure crew closes the building, safely rehouses the tenants in alternative accommodation, demolishes the building, builds a new compact one with the same tenants placed back in their rooms, and reopens. Predictable, well-tested, contained failure modes. The cost is downtime: the building is closed for the duration of the operation.

### The live rebuild crew

A different crew offers to do the same compaction without closing the building. Their method: construct a complete new building on the adjacent lot while the old one stays open. A relay team stationed at the old building's front desk intercepts every change — arrivals, departures, room reassignments — and replicates it to the new building in real time. Tenants don't relocate; they stay in the old building and go about their business, unaware that a copy of their arrangements is taking shape next door.

The new building gets its own lobby built from scratch — fresh status board, new records cabinets, new entry routes. None of the old lobby's state carries over. Everything is reconstructed from the new building's own data.

When the new building is complete and the relay has caught up, the crew swaps the address plates overnight. The old building's address now points to the new one. The old building is demolished. For tenants and guests, the transition is a brief lockout at the moment of the swap.

The advantage is real: the old building never closes during construction. The danger is also real, and it comes from the fact that two buildings are running simultaneously for the entire duration of the rebuild. The relay team has to keep up with every change in the old building and reproduce it faithfully in the new one. Resources — especially land — are roughly doubled.

The swap itself needs a moment of exclusive access: every tenant pauses, the address plates change, everyone resumes. If a guest is at the front desk mid-conversation and won't leave, the swap waits. If it waits too long, the crew gives up and the whole operation has to be attempted again.

The worst case isn't the swap failing. It's the crew getting killed mid-operation — a power failure, an administrative decision to abort, a crash. The original building survives. No tenant is harmed, no data lost. But the crew doesn't clean up after itself. The relay mechanism is still wired into the old building's front desk, adding overhead to every arrival, departure, and room change. The half-built new building sits on the adjacent lot, consuming land. Its partial records cabinets, scaffolding, and construction logs are all still there.

This is where the fire escapes get blocked. The old building is technically open, but construction equipment is piled in the corridors and the adjacent lot is occupied by a half-built shell. If land was already tight — and it often is, because the rebuild needs roughly twice the building's footprint to run — the orphaned construction can push the site past capacity. The front desk can't process new arrivals because there's nowhere to put them. Existing tenants can't rearrange their rooms. A researcher who was working happily through the construction now finds herself in a building where nothing moves — her data intact, her work untouched, but the corridors so packed with abandoned scaffolding that no one can get in or out.

The live rebuild has its place. It's the right choice when the cost of closing the building is genuinely prohibitive. But it's not the safer option just because the old building stays open. Both approaches need extra land during operation — the scheduled-closure crew also builds a new compact structure before demolishing the old one. The difference is what happens when something goes wrong. The scheduled-closure crew, interrupted, rolls back: the old building is still there, no debris, no orphaned construction. The live rebuild crew, interrupted, leaves scaffolding, triggers, half-built structures, and a site that may be harder to recover from than the downtime would have been.

### The registration emergency compliance officer

When a building approaches its registration deadline and the routine stamping pace clearly won't catch up in time, the head of housekeeping calls in the compliance officer. He's a different kind of staff member: a stern bureaucrat from the housing authority whose only job is to process registrations before deadlines. He shows up with a stamp.

He doesn't care about laundry, gardens, demographics, or the status board. He doesn't try to make the building nice. His job is keeping the building's operating license. He visits every room with unstamped tenants, stamps them permanent at maximum efficiency, ignores everything else, and leaves when the deadline is no longer in immediate danger.

The compliance officer arriving means the situation got to a state where routine housekeeping wasn't enough. That's not a failure of the housekeeping pool — it's a signal that the building's normal maintenance rhythm couldn't keep up with the arrival rate, or that something blocked routine work for long enough that pressure accumulated past safe levels. The compliance officer is a fail-safe in the engineering sense: he prevents the worst-case outcome (license revocation, campus-wide arrival shutdown) at the cost of a disruptive emergency procedure.

---

## When things go wrong

The four jobs can each fail or fall behind independently. The two failure modes that produce operational emergencies are bloat and wraparound, covered above. There are also four ways the head of housekeeping herself can fail:

**Sent home.** The campus owner has dismissed her. No dispatching happens. The housekeeper pool sits idle. Pressure accumulates on all four jobs silently across every building. Usually a configuration mistake.

**Workers can't get into rooms.** A guest at the front desk who started a multi-part query session and hasn't finished — perhaps gone for coffee with their session still active — keeps a snapshot of the building's state pinned to the moment they started. The housekeeper can't update anything that snapshot might still need to consult, because the guest expects their answer to be consistent with the state at the time they asked. An abandoned guest leaves the snapshot pinned indefinitely.

Some buildings also have an unusual arrangement: a mirror building elsewhere that reproduces the local tenant arrangement in real time, kept in sync via continuous updates shipped from the original. Nothing in actual building operations works this way — this is the one place the counterfactual stops mapping to physical reality — but it's how PostgreSQL replication works. The original promises to preserve any record the mirror might still need until the mirror confirms it has caught up, and a silent or abandoned mirror leaves the reservation open indefinitely. The housekeeper can't clear the corresponding records.

A third blocker: old registration paperwork that was prepared but never finalized — neither confirmed nor cancelled. The registrar set aside resources for it and is waiting for one signal or the other. If neither comes, the half-state persists across building cycles, holding its piece of the registration pool open.

In each case, the housekeepers keep being dispatched to buildings and being turned away.

Sometimes no single blocker is the problem. The building is simply in a busy period — a surge of arrivals and departures, heavy foot traffic in every corridor, tenants rearranging their rooms constantly. Exam season in a dormitory. The housekeepers keep being dispatched and keep getting interrupted or throttled because the cost of working alongside this much activity exceeds their limits. Nothing is broken. The building is just too loud for cleaning to happen at the pace it needs to. The system has to weather it, then catch up afterward.

**Pool too small for the work rate.** The head of housekeeping is dispatching correctly but the pool is too small or its housekeepers are too throttled to keep up. Tuning this trades catch-up speed against load on the campus's shared infrastructure.

**Status board doesn't reflect actual pressure.** The head's board tracks signals that don't capture all kinds of work. Historically, buildings with no departures (insert-only workloads) appeared idle on her board even when their stamping pressure was mounting. PostgreSQL 13 fixed this by adding signals for arrival rate, not just departure rate.

---

## Returning to the real world

The counterfactual has been a metaphor for actual PostgreSQL operations. Here's the mapping.

| Counterfactual | PostgreSQL |
|---|---|
| Campus | A PostgreSQL cluster (database server) |
| District | A database within the cluster |
| Building | A table |
| Room | A page (8KB block of heap storage) |
| Tenant | A tuple (a row version) |
| Departed tenant | A dead tuple (left by UPDATE or DELETE) |
| Records room with filing cabinets | The table's indexes |
| Front desk | The query planner |
| Front desk registry of available space per room | The free space map (FSM, one per table) |
| Attached warehouse for oversized possessions | TOAST table for oversized field values |
| Claim ticket pointing to warehouse items | TOAST pointer in the main tuple |
| Tenants-as-researchers with varied setup requirements | Variable-width tuples (text, jsonb, arrays of varying lengths) |
| Fragmented cleared zones in a room | Fragmented free space within a page |
| Lobby status board | The visibility map (one per table) |
| Settled flag | The all-visible bit |
| Completed flag | The all-frozen bit (added in 9.6) |
| Housekeeper pool | The autovacuum worker pool |
| Routine housekeeper | An autovacuum worker, or manual VACUUM |
| Aggressive housekeeper | VACUUM with vacuum_freeze_table_age exceeded, or VACUUM FREEZE |
| Head of housekeeping | The autovacuum launcher (daemon) |
| Head's status board | `pg_stat_user_tables`, `pg_stat_activity`, autovacuum work queue |
| Room turnover | Reclaiming space from dead tuples (vacuum's original function) |
| Tenant census | ANALYZE; populating `pg_statistic` for the query planner |
| Permanent registration stamping | Freezing; advancing relfrozenxid before XID wraparound |
| Campus-wide registration pool | The 32-bit transaction ID space, shared across all databases |
| Building's internal deadline | Per-table `autovacuum_freeze_max_age` threshold |
| Campus-wide arrival shutdown | Cluster refuses new transactions when XIDs approach exhaustion |
| Demolition and reconstruction crew | VACUUM FULL |
| Live rebuild crew | pg_repack, pg_squeeze (extensions) |
| New building on adjacent lot with relay team | Auxiliary table, trigger-based DML replay, and new indexes/visibility map that pg_repack builds during the rebuild |
| Address plate swap | The lock-and-rename operation at the end of pg_repack |
| Orphaned construction after interrupted rebuild | Leftover trigger, log table, and temporary table from a killed pg_repack |
| Compliance officer | The PG14 anti-wraparound failsafe vacuum |
| Guest with an open query session | A long-running transaction holding the xmin horizon |
| Mirror building and its reservation | A streaming replication replica and its slot |
| Old prepared paperwork | An abandoned two-phase commit prepared transaction |
| Sent home | `autovacuum = off` |
| Exam season | Sustained heavy write load where autovacuum cost limits throttle workers below the needed pace |
| Pool too small | `autovacuum_max_workers`, `autovacuum_vacuum_cost_limit` set too restrictively |
| Board doesn't reflect actual pressure | Pre-PG13: dead-tuple-based autovacuum triggers didn't fire on insert-only workloads, masking accumulating freezing pressure |

The four jobs map to PostgreSQL's four functions of VACUUM:

1. Room turnover → recovering disk space
2. Tenant census → updating planner statistics (ANALYZE)
3. Status board updates → updating the visibility map
4. Permanent registration stamping → preventing transaction ID wraparound (freezing)

These are four conceptually distinct operations sharing one implementation.

---

## Notes on version coverage

Written against PostgreSQL 17. A few version transitions are worth flagging where the behavior or terminology has changed materially:

- **9.4** changed the freezing mechanism. Previously, freezing replaced a tuple's xmin with a special FrozenTransactionId. Since 9.4, freezing sets a flag bit and preserves the original xmin. Some older documentation and articles still describe the pre-9.4 model; the current model is what operators will see in any recent PostgreSQL installation.
- **9.6** added the all-frozen bit to the visibility map. This allows even aggressive vacuum to skip pages that are entirely frozen, dramatically reducing the cost of anti-wraparound passes on tables that don't change.
- **13** added insert-threshold parameters (`autovacuum_vacuum_insert_threshold`, `autovacuum_vacuum_insert_scale_factor`) so autovacuum triggers on arrival rate, not just departure rate. Before this, insert-only tables accumulated freezing pressure invisibly to the autovacuum scheduler.
- **14** added the failsafe vacuum (the compliance officer in the metaphor) — when wraparound is imminent, the system drops cost limits and skips non-essential work to advance the freeze horizon as fast as possible.

## On the horizon

64-bit transaction IDs would obviate the wraparound problem entirely. Active development since approximately 2017, with formal patches submitted through commitfests from 2022 onward. Unmerged as of late 2025. Blockers: storage overhead (8 bytes per tuple, significant on narrow tables), pg_upgrade compatibility, depth of 32-bit comparison logic woven through the codebase. Commercial forks (notably Postgres Pro) have shipped 64-bit XIDs for years; upstream has not reached consensus. Operators should plan for 32-bit XID management to remain the operational reality through at least PostgreSQL 19.

A separate proposal currently on the commitfest (Antonin Houska, Cybertec) would absorb VACUUM FULL and CLUSTER into a new REPACK command, with eventual CONCURRENTLY support using logical decoding for lock-free rebuild. If accepted, bloat-recovery operations become substantially less disruptive than the current VACUUM FULL / pg_repack tradeoff.
