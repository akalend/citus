# Spec: One-Task Adaptive Executor

| Field        | Value                                 |
|--------------|---------------------------------------|
| Author       | Colm                                  |
| Branch       | `colm/one-task-adaptive-executor`     |
| Status       | Draft                                 |
| Date         | 2026-03-03                            |

---

## 1. Problem Statement

The Citus adaptive executor treats single-task queries (single-shard SELECTs, point INSERTs, single-shard UPDATE/DELETE) identically to multi-task queries. For the overwhelmingly common case of one task targeting one placement on one remote node, the executor still:

- Allocates the full `DistributedExecution` struct (with column arrays, wait-event sets, etc.)
- Creates `WorkerPool`, `WorkerSession`, `ShardCommandExecution`, `TaskPlacementExecution` objects
- Runs the full `ConnectionStateMachine` / `TransactionStateMachine` event loop
- Builds and tears down a `WaitEventSet` for a single socket
- Traverses task and placement queues designed for many-to-many fan-out

This per-query overhead is negligible for complex multi-shard queries but becomes significant at high QPS for simple point queries, which are Citus's most performance-sensitive workload.

In one sentence, the problem is that throughput of single-shard queries and OLTP workloads is negatively impacted by the overhead in the Citus Adaptive Executor. 

## 2. Goals

- [ ] Reduce per-query executor overhead for single-task, single-placement remote queries
- [ ] No behavioral change for multi-task queries (zero risk to existing functionality)
- [ ] No change to transaction semantics, error handling, or EXPLAIN output
- [ ] Measurable improvement in throughput (QPS) for simple point queries (target: at least 10%)

## 3. Non-Goals

- Optimizing multi-shard queries
- Changing the planner or fast-path router planner
- Changing connection caching / pooling behavior
- Changing INSERT...SELECT, MERGE, or repartition code paths

## 4. Background

### 4.1 Current Execution Flow
The current flow for all adaptive-executor queries, regardless of task count:

```
CitusExecScan()
  └─ AdaptiveExecutor(scanState)
       ├─ tuplestore_begin_heap()                    ← allocate result store
       ├─ CreateTupleStoreTupleDest()
       ├─ DecideTaskListTransactionProperties()      ← determine 2PC, txn blocks
       ├─ copyParamList() + MarkUnreferencedExternParams()
       ├─ CreateDistributedExecution()               ← allocate DistributedExecution
       │    ├─ palloc0 column arrays (16 cols)
       │    ├─ ShouldExecuteTasksLocally()           ← iterate placements
       │    └─ ExtractLocalAndRemoteTasks()
       ├─ StartDistributedExecution()                ← coordinated txn, shard locks
       ├─ RunDistributedExecution()                  ← THE EVENT LOOP
       │    ├─ AssignTasksToConnectionsOrWorkerPool()
       │    │    ├─ alloc ShardCommandExecution
       │    │    ├─ alloc TaskPlacementExecution per placement
       │    │    ├─ FindOrCreateWorkerPool()          ← linear scan + palloc
       │    │    └─ SortList(workerList)
       │    ├─ ManageWorkerPool()                    ← slow-start, open connections
       │    │    └─ StartNodeUserDatabaseConnection()
       │    ├─ BuildWaitEventSet()                   ← palloc + AddWaitEventToSet
       │    └─ while (unfinishedTaskCount > 0):
       │         WaitEventSetWait()
       │         ProcessWaitEvents()
       │         ConnectionStateMachine()
       │           └─ TransactionStateMachine()
       │                ├─ send BEGIN (if txn block)
       │                ├─ send query
       │                └─ ReceiveResults() → tuplestore
       ├─ FinishDistributedExecution()
       └─ SortTupleStore() (if RETURNING + ORDER BY)
```

### 4.2 Key Data Structures

| Structure                 | Allocated Per  | Purpose                              |
|---------------------------|----------------|--------------------------------------|
| `DistributedExecution`    | query          | Central execution state              |
| `WorkerPool`              | worker node    | Connection pool per node             |
| `WorkerSession`           | connection     | Per-connection state machine         |
| `ShardCommandExecution`   | task           | Per-task execution tracking          |
| `TaskPlacementExecution`  | placement      | Per-placement execution state        |
| `WaitEventSet`            | event loop     | I/O multiplexing (epoll/kqueue)      |

### 4.3 Overhead Analysis for Single-Task Queries

For a single-task, single-placement query (e.g. `SELECT * FROM t WHERE id = 42`):

1. **Struct allocation**: All 6 structs above are allocated, for exactly 1 task
2. **Event loop setup**: `WaitEventSet` built and torn down for 1 socket
3. **Worker pool management**: `FindOrCreateWorkerPool` does a linear scan of an empty list, allocates a pool, sorts a 1-element list
4. **State machine overhead**: `ConnectionStateMachine` + `TransactionStateMachine` run their full state progressions for 1 connection

### 4.4 Existing Fast Paths

- **Planner fast path** (`PlanFastPathDistributedStmt`): Speeds up planning for simple queries. Does not affect execution.
- **Local execution** (`local_executor.c`): Completely bypasses adaptive executor when the shard is on the coordinator. Uses direct PostgreSQL executor calls.
- **`fastPathRouterPlan` flag** on `DistributedPlan`: Set by the planner for single-table, single-partition-value queries. Used in `CitusBeginReadOnlyScan` and `CitusBeginModifyScan` to decide deferred pruning, but not used by the executor itself.

## 5. Proposed Design

### 5.1 High-Level Approach

The one task adaptive executor clones and refactors the AdapativeExecutor() function in adaptive_executor.c. The signature of the function is:
 
`TupleTableSlot *OneTaskAdaptiveExecutor(CitusScanState *scanState)`

The OneTaskAdaptiveExecutor is selected by the distributed planner, in `FinalizePlan()`. We will need to extend citus_custom_scan.c with a new function `CitusExecOneTaskScan()`, that is identical to `CitusExecScan()` except it calls `OneTaskAdaptiveExecutor()` instead of `AdaptiveExecutor()`. We need a new CustomExecMethods instance to be called `OneTaskAdaptiveExecutorCustomExecMethods`, with custom name "OneTaskAdaptiveExecutor" and `ExecCustomScan` method pointing to `CitusExecOneTaskScan()`. There must also be a new function `static Node* OneTaskAdaptiveExecutorCreateScan()`, that is similar to `AdaptiveExecutorCreateScan()` except that it uses `MULTI_ONE_TASK_ADAPTIVE_EXECUTOR` as the executorType and the custom scan state methods are pointing at `OneTaskAdaptiveExecutorCustomExecMethods`. This implies a new CustomScanMethods instance, let's call that `OneTaskAdaptiveExecutorCustomScanMethods`, and that must be registered by `RegisterCitusCustomScanMethods()`. 

The `MultiExecutorType` Enum in multi_server_executor.h needs a new value: `MULTI_ONE_TASK_ADAPTIVE_EXECUTOR`

The implementation of `OneTaskAdaptiveExecutor()` can be in `adaptive_executor.c`, because it may need to use existing functionality that `AdaptiveExecutor()` uses. 

`FinalizePlan()` in distributed_planner.c uses the one task adaptive executor if `JobExecutorType()` returns `MULTI_ONE_TASK_ADAPTIVE_EXECUTOR`; `JobExecutorType()` does so if the distributed plan's `fastPathRouterPlan` is true. 

### 5.2 Eligibility Criteria

**Eligibility criteria:**
- [ ] `distribtuedPlan->fastPathRouterPlan` is true
- [ ] Single placement for that task
- [ ] No dependent tasks / sub-plans that require special handling
- [ ] <!-- Add additional criteria as needed -->

**Detection point:**
Function `JobExecutorType()` detects and returns `MULTI_ONE_TASK_ADAPTIVE_EXECUTOR` if the given distributed plan has its `fastPathRouterPlan` turned on (i.e., is true). `FinalizePlan()` responds to that by using OneTaskAdaptiveExecutorCustomScanMethods for the scan's methods.

### 5.3 Optimized Execution Path

`CitusExecOneTaskScan()` is the entry point. It falls back to `AdaptiveExecutor()` for EXPLAIN ANALYZE (which needs per-task cost annotations), otherwise calls `OneTaskAdaptiveExecutor()`:

```
CitusExecOneTaskScan(node)
  ├─ if RequestedForExplainAnalyze(scanState) → AdaptiveExecutor(scanState)  ← fallback
  └─ else → OneTaskAdaptiveExecutor(scanState)
  └─ IncrementStatCounterForMyDb(STAT_QUERY_EXECUTION_SINGLE_SHARD)
  └─ ReturnTupleFromTuplestore(scanState)
```

The full flow of `OneTaskAdaptiveExecutor(scanState)`:

```
OneTaskAdaptiveExecutor(scanState)
  │
  ├─── Phase 1: Setup ───────────────────────────────────────────────────────
  │  AllocSetContextCreate("OneTaskAdaptiveExecutor")
  │  tuplestore_begin_heap()                          ← result store
  │  CreateTupleStoreTupleDest()
  │
  ├─── Phase 2: Transaction Properties & Parameters ─────────────────────────
  │  DecideTaskListTransactionProperties(modLevel, taskList)
  │    → determines useRemoteTransactionBlocks:
  │        REQUIRED  — if modifying data, or inside coordinated txn
  │        ALLOWED   — if read-only, outside explicit txn
  │        DISALLOWED — if excludeFromTransaction or CREATE INDEX CONCURRENTLY
  │    → determines requires2PC:
  │        true if multi-shard write across nodes, or reference table modification
  │  if (paramListInfo != NULL && !paramListInfo->paramFetch):
  │    copyParamList()
  │    MarkUnreferencedExternParams()
  │
  ├─── Phase 3: Coordinated Transaction & Locks ─────────────────────────────
  │  if useRemoteTransactionBlocks == REQUIRED:
  │    UseCoordinatedTransaction()
  │  if requires2PC:
  │    Use2PCForCoordinatedTransaction()
  │  AcquireExecutorShardLocksForExecution(modLevel, taskList)
  │
  ├─── Phase 4: Zero-Task Short Circuit ─────────────────────────────────────
  │  if taskList == NIL:
  │    goto finish                                    ← nothing to execute
  │
  ├─── Phase 5: Local vs. Remote Split ──────────────────────────────────────
  │  if ShouldExecuteTasksLocally(taskList):
  │    ExtractLocalAndRemoteTasks()
  │      → localTaskList, remoteTaskList
  │  else:
  │    remoteTaskList = taskList
  │
  ├─── Phase 6: Local Execution (if local) ──────────────────────────────────
  │  if localTaskList != NIL:
  │    ExecuteLocalTaskListExtended(localTaskList, ...)
  │      → uses PG local executor (ExecutorStart/Run)
  │      → results go into same tuplestore
  │      → es_processed updated for DML
  │    (skip to Phase 8)
  │
  ├─── Phase 7: Remote Execution (the core optimization) ───────────────────
  │  │
  │  ├─ 7a. Placement Lookup
  │  │    task = linitial(remoteTaskList)
  │  │    taskPlacement = linitial(task->taskPlacementList)
  │  │    LookupTaskPlacementHostAndPort() → nodeName, nodePort
  │  │
  │  ├─ 7b. Connection Acquisition
  │  │    placementAccessList = PlacementAccessListForTask(task, taskPlacement)
  │  │    if useRemoteTransactionBlocks != DISALLOWED:
  │  │      connection = GetConnectionIfPlacementAccessedInXact(...)
  │  │                   ↑ reuse connection from earlier in same txn
  │  │    if connection == NULL:
  │  │      connection = GetNodeUserDatabaseConnection(nodeName, nodePort)
  │  │      if PQstatus != CONNECTION_OK → ereport(ERROR)
  │  │
  │  ├─ 7c. Dead Connection Detection (cached connections only)
  │  │    if remoteTransaction.transactionState == REMOTE_TRANS_NOT_STARTED:
  │  │      sock = PQsocket(connection->pgConn)
  │  │      peekRc = recv(sock, &peekBuf, 1, MSG_PEEK | MSG_DONTWAIT)
  │  │        peekRc == 0        → EOF, remote closed         → dead
  │  │        peekRc > 0         → unexpected data (FATAL)    → dead
  │  │        errno != EAGAIN    → socket error               → dead
  │  │        errno == EAGAIN    → nothing pending             → alive
  │  │      if dead:
  │  │        CloseConnection(connection)
  │  │        connection = GetNodeUserDatabaseConnection(...)  ← one retry
  │  │
  │  ├─ 7d. Claim & Track
  │  │    ClaimConnectionExclusively(connection)
  │  │    AssignPlacementListToConnection(placementAccessList, connection)
  │  │
  │  ├─ 7e. 2PC for Expanded Transactions
  │  │    if TRANSACTION_BLOCKS_REQUIRED
  │  │       && XactModificationLevel == XACT_MODIFICATION_DATA
  │  │       && TaskListModifiesDatabase(...)
  │  │       && !ConnectionModifiedPlacement(connection):
  │  │      Use2PCForCoordinatedTransaction()
  │  │
  │  ├─ 7f. BEGIN Remote Transaction
  │  │    if useRemoteTransactionBlocks == REQUIRED:
  │  │      RemoteTransactionBeginIfNecessary(connection)     ← sends BEGIN
  │  │
  │  ├─ 7g. Send Query
  │  │    queryString = TaskQueryStringAtIndex(task, 0)
  │  │    binaryResults = EnableBinaryProtocol && CanUseBinaryCopyFormat(tupleDesc)
  │  │    if paramListInfo && !task->parametersInQueryStringResolved:
  │  │      ExtractParametersForRemoteExecution(...)
  │  │      SendRemoteCommandParams(connection, queryString, params, binaryResults)
  │  │    else:
  │  │      SendRemoteCommand(connection, queryString)        ← text mode
  │  │      (or SendRemoteCommandParams with binaryResults)   ← binary mode
  │  │    if querySent == 0 → ereport(ERROR)
  │  │
  │  ├─ 7h. Enable Single-Row Mode
  │  │    PQsetSingleRowMode(connection->pgConn)
  │  │    if fails → UnclaimConnection(), ereport(ERROR)
  │  │
  │  ├─ 7i. Build Result Metadata
  │  │    if tupleDescriptor != NULL:
  │  │      attInMetadata = binaryResults
  │  │        ? TupleDescGetAttBinaryInMetadata(tupleDesc)
  │  │        : TupleDescGetAttInMetadata(tupleDesc)
  │  │    columnArray = palloc0(columnCount * sizeof(void *))
  │  │    if binaryResults:
  │  │      stringInfoDataArray = palloc0(columnCount * sizeof(StringInfoData))
  │  │
  │  ├─ 7j. Result Loop (simple poll — no WaitEventSet, no state machines)
  │  │    rowContext = AllocSetContextCreate("RowContext")
  │  │    while (!fetchDone):
  │  │      │
  │  │      ├─ if PQisBusy(connection):
  │  │      │    WaitLatchOrSocket(MyLatch,
  │  │      │      WL_SOCKET_READABLE | WL_LATCH_SET | WL_EXIT_ON_PM_DEATH,
  │  │      │      sock, 0, PG_WAIT_EXTENSION)
  │  │      │    ResetLatch(MyLatch)
  │  │      │    CHECK_FOR_INTERRUPTS()
  │  │      │    if socket readable: PQconsumeInput()
  │  │      │    continue
  │  │      │
  │  │      ├─ result = PQgetResult(connection)
  │  │      │    result == NULL → fetchDone = true; break
  │  │      │
  │  │      ├─ switch PQresultStatus(result):
  │  │      │    PGRES_COMMAND_OK    → rowsProcessed += PQcmdTuples; continue
  │  │      │    PGRES_TUPLES_OK    → final marker after single-row mode; continue
  │  │      │    PGRES_SINGLE_TUPLE → process rows (below)
  │  │      │    other              → ReportResultError(connection, result, ERROR)
  │  │      │
  │  │      └─ for each row in result:
  │  │           MemoryContextSwitchTo(rowContext)
  │  │           build columnArray from PQgetvalue / PQgetisnull
  │  │           heapTuple = binaryResults
  │  │             ? BuildTupleFromBytes(attInMetadata, columnArray)
  │  │             : BuildTupleFromCStrings(attInMetadata, columnArray)
  │  │           tupleDest->putTuple(tupleDest, task, heapTuple)
  │  │           MemoryContextReset(rowContext)
  │  │           rowsProcessed++
  │  │
  │  │    MemoryContextDelete(rowContext)
  │  │
  │  └─ 7k. Release Connection & Count Rows
  │       UnclaimConnection(connection)
  │       if commandType != CMD_SELECT:
  │         es_processed += rowsProcessed
  │
  ├─── Phase 8: Finish ─────────────────────────────────────────────────────
  │  finish:
  │  if TaskListModifiesDatabase(modLevel, taskList):
  │    XactModificationLevel = XACT_MODIFICATION_DATA
  │  MemoryContextSwitchTo(oldContext)
  │  return NULL                                      ← results in tuplestore
  │
  └─── (COMMIT/ROLLBACK of remote txn handled by coordinator at txn end)
```

**Key differences from `AdaptiveExecutor()` (section 4.1):**

| Aspect | AdaptiveExecutor | OneTaskAdaptiveExecutor |
|---|---|---|
| Struct allocation | `DistributedExecution`, `WorkerPool`, `WorkerSession`, `ShardCommandExecution`, `TaskPlacementExecution` | None of these — direct variables |
| Event loop | `WaitEventSet` + `BuildWaitEventSet` + `ProcessWaitEvents` | `WaitLatchOrSocket()` on a single socket |
| Connection management | `FindOrCreateWorkerPool` → linear scan, `SortList`, slow-start | Direct `GetNodeUserDatabaseConnection` or in-txn reuse |
| State machines | `ConnectionStateMachine` + `TransactionStateMachine` (multi-state FSM) | Linear sequential flow — no state tracking |
| Dead connection handling | Implicit via state machine retries and pool management | Explicit `recv(MSG_PEEK)` probe + one retry |
| EXPLAIN ANALYZE | Inline, with per-task annotations | Falls back to full `AdaptiveExecutor` |
| Dependent jobs | `ExecuteDependentTasks()` before main execution | Not supported (ineligible by design) |
| Multi-row INSERT | Supported | Ineligible (detected by `JobExecutorType`) |

**Edge cases within the execution flow:**

| Scenario | Behavior |
|---|---|
| Zero tasks (empty taskList after pruning) | Short-circuits at Phase 4, no connection opened |
| Local shard (shard on coordinator) | Handled in Phase 6 via `ExecuteLocalTaskListExtended`; no remote connection |
| Explicit transaction block (`BEGIN...COMMIT`) | `useRemoteTransactionBlocks = REQUIRED`; BEGIN sent in 7f; same connection reused for subsequent statements; COMMIT at txn end |
| Prepared statement / parameterized query | Parameters extracted in 7g via `ExtractParametersForRemoteExecution`; sent with `SendRemoteCommandParams` |
| RETURNING clause on DML | Results processed like SELECT rows in 7j; rowsProcessed also counted from `PQcmdTuples` |
| Binary protocol eligible | Detected in 7g; binary metadata built in 7i; `BuildTupleFromBytes` in 7j |
| Cached connection found dead | Detected in 7c via non-blocking `recv(MSG_PEEK)`; closed and retried once. If retry also fails, `ereport(ERROR)` |
| Connection failure (new connection) | `ereport(ERROR)` immediately — no fallback to other placements or to `AdaptiveExecutor` |
| Query send failure | `ereport(ERROR)` after `UnclaimConnection` |
| Remote error during result fetch | `ReportResultError(connection, result, ERROR)` — same error as `AdaptiveExecutor` |
| Shard moved / split (shard absent) | Executor errors out — shard existence is assumed from planner |
| Cancel / interrupt | `CHECK_FOR_INTERRUPTS()` in result loop; standard PG cancellation |

### 5.4 Connection Handling

Connection handling should be equivalent to the connection handling code-path for a single task execution in AdaptiveExecutor, but without the overhead for multiple tasks, sessions and connections. There is at most one connection, if the shard is remote. If the execution is local, then there are no connections needed.

### 5.5 Transaction Semantics

Transaction semantics should be identicial to a single task execution in AdaptiveExecutor, without multi-task overhead.

### 5.6 Error Handling

Error handling should be identical to a single task execution in AdaptiveExecutor. If an error is hit, do not fall back to AdaptiveExecutor, just error out, with the same message used by AdaptiveExecutor.

### 5.7 Result Handling

Use a tuplestore, just like AdaptiveExecutor currently does. This may be enhanced to a result slot in the future, if the query returns provably one row, or if it is a modification statement without a RETURNING clause.

## 6. Affected Files

citus_custom_scan.h
citus_custom_scan.c
adaptive_executor.c
local_executor.c
distributed_planner.c

## 7. Testing Strategy

### 7.1 Correctness

- [ ] All existing regression tests pass without modification

### 7.2 Edge Cases

- [ ] Single-shard SELECT with parameters (prepared statement)
- [ ] Single-shard INSERT (fast-path + deferred pruning)
- [ ] Single-shard UPDATE/DELETE
- [ ] Query in an explicit transaction block (`BEGIN ... COMMIT`)
- [ ] Query with `force_delegation`
- [ ] Query during shard move / split (shard doesn't exist → reroute)
- [ ] Query with RETURNING clause
- [ ] EXPLAIN ANALYZE on optimized path
- [ ] Connection failure during optimized execution
- [ ] Fallback from optimized path to full executor

### 7.3 Performance

- [ ] Benchmark: pgbench / custom harness with simple point queries
- [ ] Metric: QPS at saturation, p99 latency
- [ ] Comparison: before vs. after on same hardware

## 8. Rollout & Risk

### 8.1 Feature Flag / GUC

The GUC `citus.enable_single_task_fast_path` enables or disables this feature. It is checked by `JobExecutorType()`

| GUC Name | Default | Description |
|----------|---------|-------------|
| <!-- e.g. `citus.enable_single_task_fast_path` --> | <!-- on --> | <!-- ... --> |

### 8.2 Risks

- [ ] Behavioral divergence between fast path and full executor
- [ ] Edge case where eligibility check is wrong (query uses fast path but shouldn't)
- [ ] <!-- ... -->

### 8.3 Rollback Plan

## 9. Future Work

- [ ] Avoid tuplestore for single-row results (direct slot return)
- [ ] Pre-allocated single-task execution state (avoid palloc per query)
- [ ] <!-- ... -->

## 10. Open Questions
