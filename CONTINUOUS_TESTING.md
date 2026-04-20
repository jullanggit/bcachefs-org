## Relevant properties
- clearly possible shrinks should always succeed and not be unnecessarily stalled
- clearly impossible shrinks should fail reasonably fast
- previous/in-progress resizes should not influence new resizes beyond a short additional delay
- TBD

## Design
overnight-style/continuous test
- produce a log file with all operations, and clearly greppable errors
  - errors
    - shrink failed when it should succeed and vice-versa
    - general fs invariant violation (probably most easily checked through fsck)
- property-based testing?
- permanent/concurrent fs activity
  - writes, reads, deletes
  - resizes
  - device additions/removals
  - replication/other fs option changes
- one thread that synchronously kicks off and manages tasks (that may be asynchronous / overlapping)
- validates that at all points the fs state/task outcomes are what they are expected to be, based on TBD
