# Step Functions orchestration of parallel Pyomo candidates on AWS Lambda

## Problem and agreed scope

Evaluate independent LP/MILP candidates with HiGHS, with one candidate per
Lambda invocation. Candidates must complete comfortably within the standard
Lambda 900-second invocation limit, including model construction and I/O.
Support more than 40 concurrent candidates using Step Functions Distributed
Map from the outset.

AWS Step Functions owns workflow composition, candidate fan-out/fan-in,
retry policy, and batch completion. Lambda owns individual candidate evaluation.
The submission client starts and monitors a batch; it does not dispatch each
candidate itself or keep an orchestration Lambda running while solves finish.

Deliver a self-contained AWS SAM example, a replaceable demonstration model,
Linux container packaging, tests, and deployment instructions. Do not modify
Pyomo core, add AWS dependencies to the main package, deploy resources, or
spend AWS money as part of implementation.

## Repository evidence

- There is no existing Lambda deployment or AWS orchestration implementation.
- [The native HiGHS interface](pyomo/contrib/solver/solvers/highs.py) provides
  the solver integration.
- [Solver configuration](pyomo/contrib/solver/common/config.py) supports threads,
  time limits, explicit solution loading, and nonoptimal-result handling.
- [Solver results](pyomo/contrib/solver/common/results.py) distinguish
  termination reasons from feasible/optimal/no-solution states.
- Keep the integration under `examples/pyomo/aws_lambda/`; keep its Python
  dependencies and infrastructure isolated from the main Pyomo package.

## Architecture

```text
Python submission client
  -> immutable S3 batch manifest and candidate/shared-data objects
  -> Step Functions Standard workflow
     -> ValidateBatch: validate manifest metadata and execution settings
     -> EvaluateCandidates: Distributed Map reads the S3 manifest
        -> one Standard child workflow per candidate
        -> EvaluateCandidate: synchronous Lambda task
           -> validate candidate and fetch versioned data
           -> build a fresh Pyomo model and HiGHS solver
           -> solve and persist a structured result in S3
        -> bounded Retry for transient invocation failures
        -> Catch -> RecordCandidateError Lambda on terminal failure
     -> S3 ResultWriter exports compact child outcomes
     -> SummarizeBatch: bounded summary/reconciliation Lambda
     -> BatchOutcome: Choice -> completed / partial failure / incomplete
  -> durable batch summary and candidate-result references
```

Use Standard child workflows, not Express: Express's duration limit is too
short for the agreed candidate envelope. Do not batch multiple candidates
into one worker invocation. Parallelism is across Lambda invocations, not
threads or multiprocessing inside a worker.

## Step Functions composition

### Parent workflow

1. `ValidateBatch` checks the batch contract, manifest reference, positive
   concurrency cap, and supported model/solver settings. The submission client
   validates candidate ID uniqueness before publishing the immutable manifest;
   validation must not load a large manifest into Step Functions state.
2. `EvaluateCandidates` uses an S3 `ItemReader`, a Distributed Map
   `ItemProcessor`, and `ProcessorConfig` with `Mode: DISTRIBUTED` and
   `ExecutionType: STANDARD`. Use `ItemSelector` to pass each candidate and
   its batch context. Do not configure `ItemBatcher`.
3. Set `MaxConcurrencyPath` from the validated batch configuration and export
   compact child outcomes through S3 `ResultWriter`. Full solutions remain in
   candidate-specific S3 objects rather than accumulating in workflow state.
4. `SummarizeBatch` runs only after the Map finishes and reconciles its outcomes
   against the expected candidate manifest. Produce status counts and result
   references; ranking/selecting candidates is outside the initial scope.
5. `BatchOutcome` uses a `Choice` state to distinguish completed evaluations,
   candidate execution failures, and missing results. A handled candidate error
   can complete its child workflow without making the batch business outcome
   successful. Infeasibility is a solver outcome, not an infrastructure failure.
6. Catch Map-level infrastructure failures separately, persist a failed-batch
   record when possible, and end visibly in `Fail`. Do not retry the entire Map
   automatically: that can rerun candidates that have already completed.

### Per-candidate child workflow

- `EvaluateCandidate` uses the optimized `arn:aws:states:::lambda:invoke`
  integration with synchronous request/response behavior. Step Functions waits
  for the result; do not use asynchronous `InvocationType: Event`.
- Pass only the candidate contract and S3 references. Select the compact Lambda
  response payload, not the integration's full response metadata.
- Keep model construction and solving in the same worker invocation. Do not
  pass an in-memory model between separate Lambda tasks.
- Define named transient Lambda service/throttling retries at the task level.
  Route exhausted retries, worker exceptions, and task timeouts to
  `RecordCandidateError`; deterministic input errors are not retried.
- Give the task timeout headroom beyond the worker's configured timeout so
  Lambda's own timeout can be observed. A task timeout must not be interpreted
  as proof that underlying computation stopped; conditional terminal publication
  must also handle a late worker result racing with an error record.
- Return a durable candidate outcome reference from either successful evaluation
  or successful error recording. If error recording itself fails, fail the child
  visibly instead of pretending a candidate result exists.

Define this composition declaratively in Amazon States Language (ASL), deployed
through AWS SAM. Grant the state machine only the Lambda invocation, S3
manifest/result, and child execution permissions it needs, including the
Distributed Map permissions to start and inspect child executions.

## Contracts and behavior

- Submission includes a unique batch ID, unique candidate IDs, model version,
  candidate parameters, immutable shared-data references, solver time budget,
  and requested concurrency. Validate duplicates before starting evaluation.
- Send JSON data or S3 references, not pickled Pyomo models, solver objects,
  arbitrary Python code, or user-selected import paths.
- Store large inputs and outputs in S3. Keep state-machine payloads below
  256 KiB, even though synchronous Lambda invocation permits larger payloads.
- Build a fresh model and solver per invocation. Reuse only AWS clients and
  verified immutable cached data across warm invocations.
- Results include batch/candidate identity, input fingerprint, model/solver
  version, termination condition, solution status, objective if feasible,
  bound/gap when available, elapsed time, and a solution artifact reference.
  Normalize unavailable/nonfinite values explicitly to valid JSON.
- Load/evaluate variables and objective only when a feasible solution exists.
  Preserve infeasible, unbounded, and time-limited outcomes as explicit domain
  results. A time-limited feasible incumbent is not reported as optimal.
- Invalid inputs and infrastructure/runtime failures are explicit error
  outcomes, never numerical penalty values or success-shaped fallbacks.
- Use deterministic, fingerprinted result keys and conditional publication of
  terminal records. A retry can reuse an already committed matching result.
  Do not promise exactly-once computation: duplicate execution remains possible.
- Retry only named transient AWS failures with bounded exponential backoff
  and jitter. Do not retry deterministic validation/model failures.
- A child Catch path persists candidate-level terminal errors, including worker
  timeouts. Failure to persist an error fails visibly. Reconcile all expected
  candidate IDs; missing outcomes make a batch incomplete, not successful.
- Publish batch status and counts explicitly; retain partial results when some
  candidates fail. Infrastructure failure of the Map itself must remain visible.

## Resource and operational controls

- Expose a positive Distributed Map MaxConcurrency parameter; use 50 as an
  initial example setting, subject to available account/Region quota.
  Never rely on Distributed Map's uncapped default fan-out.
- Set worker reserved concurrency to bound aggregate compute across simultaneous
  batches; leave capacity for summary/error handling and unrelated functions.
  Map concurrency is per execution, not a global account limiter.
- Start HiGHS with one solver thread per worker. Benchmark memory/CPU allocations
  before increasing thread count or concurrency.
- Configure a 900-second worker timeout and a smaller solver budget capped by
  the remaining invocation time after model construction, with an explicit
  I/O/cleanup margin. Reject invocation budgets that cannot leave that margin.
  Solver limits are not a substitute for the Lambda hard deadline.
- Package the checked-out Pyomo version, pinned compatible highspy dependencies,
  and AWS SDK in an AWS Python Linux container for one explicit architecture
  (initially x86_64). Windows binaries are not deployable to Lambda.
- Use least-privilege IAM, encrypted private S3 storage, explicit log retention,
  and structured identity/status/timing logs without full candidate data.
- CloudWatch alarms/metrics cover errors, throttling, duration, memory, and
  incomplete batches. Document transition, compute, storage, and retry costs.
- If candidate runtime or memory exceeds ordinary Lambda limits, use AWS Batch
  or ECS/Fargate workers instead; that backend is outside this implementation.

## Implementation todos

1. Define candidate/result contracts and the replaceable LP/MILP model builder.
   Add a sequential evaluator as the reference for correctness comparisons.
2. Implement the Lambda evaluator, terminal-error recorder, durable result
   publication, and bounded summary/reconciliation behavior. Bound summary
   work explicitly and report oversized batches rather than timing out silently.
3. Add the container and AWS SAM infrastructure: S3, worker/support functions,
   Standard Distributed Map workflow, ItemReader/ResultWriter, IAM, configurable
   concurrency, retries/catches, logging, and alarms. Implement the explicit
   parent and child states above; expose the state machine ARN and results bucket
   as deployment outputs.
4. Add the submission/results client and sample inputs. Upload an immutable
   manifest, start one workflow execution per batch, and expose status and
   S3 result locations. Do not keep a client connection open for every solve.
5. Add isolated tests and document local execution, Linux container build,
   explicit opt-in deployment, submitting batches, replacing the model, tuning,
   failure interpretation, costs, and targeted teardown.

Likely example files: `README.md`, `template.yaml`, `workflow.asl.json`,
`Dockerfile`, `requirements.txt`, `model.py`, `contracts.py`, `worker.py`,
`results.py`, `submit.py`, sample JSON inputs, and `tests/`.
Choose exact module boundaries during implementation without changing the scope.

## Validation

- Unit-test schema validation, duplicate IDs, result serialization, infeasible
  and unbounded cases, time limits with/without incumbents, remaining-budget
  calculation, warm-worker isolation, retry publication, and missing outcomes.
- Mock AWS I/O for dependency-light tests; never require credentials in default
  tests. Keep solver-dependent tests appropriately marked and skip explicitly
  when HiGHS is unavailable.
- Compare a known batch's sequential and worker-evaluated objectives and statuses.
- Test workflow composition: one candidate per child invocation, Standard child
  execution, configured concurrency, compact payload selection, named transient
  retries, terminal-error recording, Map-level failure handling, and batch outcome
  classification. Verify a candidate error does not silently become batch success.
- Validate SAM and ASL using their existing tools if available. Build and invoke
  the Linux container locally to verify native HiGHS imports and a real solve.
  Explicitly report unavailable validation tools; do not claim deployment proof.
- Run targeted pytest and Black checks on the example only.
- Document a separate, user-authorized AWS smoke test: submit over 40 candidates,
  demonstrate overlapping worker invocations, enforce the configured concurrency
  cap, produce one canonical result per candidate, and reconcile all outcomes.
  Exercise transient failure, timeout/error recording, and partial batch failure.
- No cloud deployment or cloud smoke test runs without explicit authorization.

## References

- https://docs.aws.amazon.com/step-functions/latest/dg/state-map.html
- https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html
- https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html
- https://docs.aws.amazon.com/lambda/latest/dg/images-create.html
