### Housekeeping

PostgreSQL's `VACUUM` is one command that does three unrelated things — reclaiming space, maintaining the visibility map, and freezing transaction IDs against wraparound — named after only one of them. Run as a background daemon (autovacuum), it also performs a fourth separate job which updates planner statistics. 

By suggesting cleanup, `VACUUM` by itself misdirects naive problem solvers in situations where reclaiming space is not the issue at hand. Documents in this repo build a univeristy campus housekeeping metaphor that intend to make the four functions legible to facilitate discriminatory mental modeling.

*quixote_time_travel.md* — the metaphor, as a story. A time-traveling DBA explains to Michael Stonebraker, the computer scientist behing the original academic research POSTGRES project that became PostgreSQL a decade later and which had a since-removed time-travel feature, what the `VACUUM` command became over forty years, and why it should have been called HOUSEKEEPING.

*housekeeping_mapping.md* — the campus model mapped to the actual system: the four functions, the autovacuum dispatcher and workers, the failsafe vacuum, manual vacuum (full or repack), the failure modes, and the configuration parameters in campus terms.

*housekeeping_haas_playbook.md* — operational troubleshooting in the campus model, organized around the ways autovacuum fails: slow or stuck, spinning without progress, skipped, and starved.
