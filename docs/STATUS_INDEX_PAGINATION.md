# Status-index pagination

Status indexes are compacted as invoices leave a status. A page may therefore
contain fewer entries than its requested size when historical slots are stale.
Clients must advance the returned cursor and continue requesting pages until
the cursor is exhausted; a short page is not an end-of-list signal.
