2-phase commit
==============

# Differences from last paper

* No applications
* Not a novel product - optimization on existing stuff. Has a nice background of existing
  algorithms
* Lampson was an early Turing award winner - older paper

# details

* When can you talk about your state?
  * only after you commit something to logs.
  * External observer cannot tell if you crashed and recovered before sending a message
  * **observability** - hugely important. I can do whatever I want until you can observe me.
  * Recovery state after crashing does not have to be the same as the original state - but it does
    have to be compatible.
* How do you force a write?
  * fsync a file. When one of them returns you know it is on disk
  * disks can write sectors atomically. But what if it's multiple sectors?
    * write onto a temporary file and then do an atomic rename to replace the original file.
    * Alternatively store timestamps of each file, algo knows to read newest one. Keep checksum to
      deal with multiple sectors
* Log write format
  * should be idempotent - for instance, what if logs write things like "create /a/b/c"?
    * not idempotent, if it crashes once to create /a/b/c and tries again it can crash again as
      soon as it recovers
  * Instead try just writing values of individual blocks, which can be done idempotently.
    * This can be much less space efficient than above but it's more reliable.
    * How to deal with log size?
      * checkpointing
      * if everything is flushed to disk you can delete the log

# assumed simplifications

* Relies on a single master - if it dies you're screwed
  * electing masters can be tricky - what if some think the masters died but it didn't? do you end
    up with 2 masters?
* also people can lie...

# pseudocode exercise for PrN

**NOTE:** ACK phase is just for garbarge collection!

**NOTE:** This is not official! This is just what I came up with.

proposed coordinator code:

```
func server(tid, cohort) {
  forceWriteToLog(tid, PREPARE, cohort)
  send(tid, PREPARE, cohort)

  votes = waitForVotes()

  if (votes.contains(ABORT_VOTE) || votes.contains(TIMEOUT)) {
    forceWriteToLog(tid, ABORT, cohort)
    sendToAll(ABORT)
  } else {
    forceWriteToLog(tid, COMMIT, cohort)
    sendToAll(COMMIT)
  }

  waitForAcks()
  writeToLog(tid, END)
}
```

cohort code:

```
func cohort() {
  tid = waitForPrepare()

  result = computeResult()
  forceWriteToLog(tid, result)
  if (result == SUCCESS) {
    sendCommitVote()
  } else {
    sendAbortVote()
  }

  coordinator_result = waitForResult()

  logTransaction(result)
  if (coordinator_result == COMMIT) {
    forceCommit(result) # actually do the transaction
    sendCommitAck()
  } else {
    sendAbortAck()
  }
}
```
