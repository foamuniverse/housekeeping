# Housekeeping

PostgreSQL's `VACUUM` is one command that does three unrelated things — reclaiming space, maintaining the visibility map, and freezing transaction IDs against wraparound — named after only one of them. The autovacuum daemon runs it automatically, along with a fourth job, `ANALYZE`, which updates the planner's statistics. Four jobs, one misleading name: "vacuum" says cleanup, but three of the four aren't cleanup, and when something goes wrong the name points you at a space-reclaiming function that might have nothing to do with the problem. This repo builds a university-campus housekeeping metaphor that gives each job its own name, so the four stop collapsing into one and reasoning about them comes more naturally.

*quixote_time_travel.md* — the metaphor, as a story. A time-traveling DBA explains to Michael Stonebraker, the computer scientist behind the 1980s Berkeley POSTGRES research project that became PostgreSQL a decade on and had a since-removed time-travel feature, what the `VACUUM` command evolved into over forty years, and why it should have been called HOUSEKEEPING.

*housekeeping_mapping.md* — the campus model mapped to the actual system: the four functions, the autovacuum dispatcher and workers, the failsafe vacuum, manual vacuum, rebuild operations, the failure modes, and the configuration parameters in campus terms.

*housekeeping_haas_playbook.md* — operational troubleshooting in the campus model, organized around the ways autovacuum fails: slow or stuck, spinning without progress, skipped, and starved.
