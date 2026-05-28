# @computesdk/bench

Tinybench-style benchmarking toolkit for ComputeSDK.

`@computesdk/bench` runs warmups + measured iterations across a suite of named tasks and emits structured benchmark events.

## Installation

```bash
npm install @computesdk/bench
```

## Quick Start

```ts
import { createBench } from '@computesdk/bench';
import { compute } from 'computesdk';

const bench = createBench({
  label: 'sandbox-lifecycle',
  groupId: 'burst-100k-e2b',
  shard: { index: 0, count: 100 },
  captureOutput: { file: '/root/run.log' },
});

bench.add('create', async (ctx) => {
  ctx.log('booting sandbox...');
  const sandbox = await compute.sandbox.create();
  ctx.log(`ready: ${sandbox.sandboxId}`);
});

bench.add('exec', async (ctx) => {
  ctx.log('running command');
  // ...
});

bench.add('destroy', async (ctx) => {
  // ...
});

const result = await bench.run({
  iterations: 25,
  warmup: 3,
  provider: 'e2b',
});

for (const task of result.tasks) {
  console.log(`${task.taskName}: p95=${task.stats.p95Ms}ms`);
}
```

## API

### `createBench(config)`

| Field | Type | Description |
|-------|------|-------------|
| `label` | `string` | Human-readable suite name |
| `apiUrl` | `string?` | Optional URL to upload benchmark events |
| `apiKey` | `string?` | Optional bearer token used when `apiUrl` is set |
| `groupId` | `string?` | Shared logical benchmark group id for multi-process/sharded runs |
| `shard` | `{ index: number; count: number }?` | Shard metadata for this process. `index` is zero-based |
| `onEvent` | `function?` | Local debug hook called with every emitted event |
| `captureOutput` | `{ file: string; flushInterval?: number }?` | Tail a process log file and emit `benchmark.output` events. `flushInterval` is milliseconds and defaults to `30000` |

Returns `{ add, run }`.

### `bench.add(taskName, fn)`

Register a named task. `fn` receives a `BenchContext`:

| Field | Type | Description |
|-------|------|-------------|
| `ctx.iteration` | `number` | Current iteration index within the current phase (0-based) |
| `ctx.phase` | `'warmup' \| 'measured'` | Current phase |
| `ctx.taskName` | `string` | Name of the current task |
| `ctx.log(...args)` | `function` | Attach a log message to this iteration's benchmark span |

### `bench.run(options?)`

| Option | Default | Description |
|--------|---------|-------------|
| `iterations` | `25` | Measured iterations per task |
| `warmup` | `3` | Warmup iterations before measuring |
| `provider` | — | Provider name tag applied to all spans |
| `throwOnError` | `true` | Re-throw the first error encountered |

Returns a `BenchSuiteResult`:

```ts
{
  label: string;
  suiteId: string;
  tasks: Array<{
    taskName: string;
    successes: number;
    failures: number;
    stats: { count, minMs, meanMs, medianMs, p95Ms, maxMs };
  }>;
}
```

## Event Hierarchy

Each `bench.run()` emits:

```
benchmark.suite  — once per run (label, tasks, provider, iterations, warmup)
  benchmark.span — once per iteration per task (timing, status, logs)
  benchmark.output — captured process output lines when captureOutput is configured
```

All spans carry `suiteId` and `suiteLabel` so results can be grouped per run.
For sharded runs, set the same `groupId` on each process and unique `shard.index` values so the API can aggregate multiple `suiteId`s into one logical benchmark.

## Notes

- Network upload only happens when `apiUrl` is configured.
- Network uploads are batched in the background with one in-flight request, and API failures do not fail the benchmark.
- `ctx.log()` messages are attached to span events and redacted.
- `captureOutput.file` tails from the beginning of the file, flushes periodically, and flushes once more when the run exits.
- This package does not capture env vars or payload content.
