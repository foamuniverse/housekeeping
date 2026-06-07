### Housekeeping

PostgreSQL's VACUUM is one command that does three unrelated things — reclaiming space, maintaining the visibility map, and freezing transaction IDs against wraparound — named after only one of them. Run as a background daemon (autovacuum), it also performs a fourth, separate job: ANALYZE, which updates planner statistics. The name points operators at cleanup when cleanup is often not the problem.

These documents build a campus metaphor that makes the four functions legible.

*quixote_time_travel.md* — the metaphor, as a story. A time-traveling DBA explains to Michael Stonebraker, the inventor of the original POSTGRES system that became PostgreSQL a decade later and which had a since-removed time-travel feature, what the VACUUM command became over forty years, and why it should have been called HOUSEKEEPING.

*housekeeping_mapping.md* — the campus model mapped to the actual system: the four functions, the autovacuum dispatcher and workers, the failsafe vacuum, manual vacuum (full or repack), the failure modes, and the configuration parameters in campus terms.

*housekeeping_haas_playbook.md* — operational troubleshooting in the campus model, organized around the ways autovacuum fails: slow or stuck, spinning without progress, skipped, and starved.
