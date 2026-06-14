# TODO


`housekeeping_vs_undo.md` — why in-place version retention beats undo-log reconstruction in specific failure modes like rollback, crash recovery, snapshot longevity, in campus vocabulary. The rival mechanism keeps a circular ledger and reconstructs any past state by reading it backward; describe its day-to-day advantages fairly, then let the failure modes speak. Deadpan; don't name Oracle, don't editorialize.

`mandrill_sentry_wraparound.md` — Mandrill (Mailchimp's transactional-email product) hit wraparound in February 2019; consult Mailchimp's postmortem writeup (which links to Sentry's hitting it in 2015) to recast the stories in housekeeping vocabulary.

`continental_rewind_op.md` — a pg_resetwal wraparound abuse story of winding the XID counter past the shutdown instead of vacuuming in Dashiell Hammett's hardboiled Continental Op style: breaking into the registration office and winding the number dispenser by hand — no one stamped, every tenant's age now measured against a forged counter, and the office's ledger of which registrations were ever finalized pointing at shredded pages.
