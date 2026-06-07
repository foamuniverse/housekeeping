# TODO


`housekeeping_vs_undo.md` — why in-place version retention beats undo-log reconstruction in specific failure modes like rollback, crash recovery, snapshot longevity, in campus vocabulary. The rival mechanism keeps a circular ledger and reconstructs any past state by reading it backward; describe its day-to-day advantages fairly, then let the failure modes speak. Deadpan; don't name Oracle, don't editorialize.
