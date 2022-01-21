# Articles

# Papers

# Architecture

## Parallel Commit
Tx commits using 2PC, but Tx can return the success to client after Phase I immediately and let background service to finish Phase II. Note the lock is still released after Phase II. Hence, the latency is reduced, but throughput is not increased too much since locking time is not optimized.

Phase II will asynchronously resolve the write intents. Write intent contains a pointer to transaction record to know the state the transaction.
