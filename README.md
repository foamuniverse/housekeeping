*There are only two hard problems in computer science: Cache invalidation, naming things, and off-by-one errors.*

—Martin Fowler popularizing Leon Bambrick's riff on Phil Karlton's famous quote


# Cache Invalidation, Naming, Off-by-One Errors and PostgreSQL's VACUUM

Databases are all about shuffling data across a speed hierarchy: CPU caches to main memory to heap to write-ahead-logging to archive and all the way back — `VACUUM` in Michael Stonebraker's original 1980s Berkeley Postgres research prototype conception a pure instance of the tail end of that hierarchy: the no-overwrite storage model as an endless persistence cache, with a metaphorical vacuum cleaner sweeping obsoleted rows from expensive live onto archival storage.

The improbably convergent historical twist dovetailing precisely onto Karlton and Bambrick is then how an originally intutively named command suggesting cleanup drifted over the next four decades to become the perfect illustration of the difficulties of cache invalidation, naming and counting things in computing: First transaction ID freezing layered onto space reclaiming, then visibility map maintenance shared between both functions and `ANALYZE` added to a daemon which performs the first, second, third, and then another, all of which got named after the first. Do we count it as two, three, or four functions misleadingly lumped under a vacuum cleaner metaphor?



# Housekeeping

PostgreSQL's `VACUUM` is one command that does two unrelated things — reclaiming space and freezing transaction IDs against wraparound — named after only one of them, while maintaining a shared data structure (the visibility map) as it goes. The autovacuum daemon runs it automatically, along with another unrelated job, `ANALYZE`, which updates the planner's statistics. Different functions, one misleading name: "vacuum" says cleanup, but only one of them is cleanup, and when something goes wrong the name points you at a space-reclaiming function that might have nothing to do with the problem. This repo builds a university-campus housekeeping metaphor that gives each job its own name, so they stop collapsing into one and reasoning about them comes more naturally.

The idea grew out of years of explaining (auto)vacuum to stakeholders of varying technical depth. Like: why does an insert-only table still need vacuuming? Nothing gets deleted, so under the cleanup picture nothing looks dirty; yet let it go unvacuumed for long enough and PostgreSQL refuses writes cluster-wide until an emergency vacuum catches up. If you think of vacuum as only dead row cleanup, this would come as an ugly surprise.

*quixote_time_travel.md* — the metaphor, as a story. A time-traveling DBA explains to Michael Stonebraker, the computer scientist behind the 1980s Berkeley POSTGRES research project that became PostgreSQL a decade on and had a since-removed time-travel feature, what the `VACUUM` command evolved into over forty years, and why it should have been called HOUSEKEEPING.

*housekeeping_mapping.md* — the campus model mapped to the actual system: the autovacuum launcher and its workers, the failsafe vacuum, manual vacuum, the failure modes, and the configuration parameters in campus terms.

*housekeeping_haas_playbook.md* — operational troubleshooting in the campus model, organized around the ways autovacuum fails: slow or stuck, spinning without progress, skipped, and starved.

Note on audience/reception: The metaphorical machinery to recast `VACUUM` as HOUSEKEEPING is heavy, and performing the mental remap isn't free. People fluent in PostgreSQL internals have built the structure the campus vocabulary scaffolds; to them it will read as irritating or frivolous, and advice to novices — "you have to learn the real terms anyway" — is correct. That's why every campus term is introduced with the real term beside it, and the playbook names the real parameters and system views throughout: the campus layer carries the relationships the real terms fail to convey.
