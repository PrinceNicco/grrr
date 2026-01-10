Problem Setup
Review the problem environment and setup instructions before creating your solution.

Parallel task execution

Implement a parallelTasks() function that enables executing multiple CLI tasks concurrently with configurable concurrency limits. This feature allows users to run independent operations in parallel while maintaining control over resource usage and error handling behavior.

Export parallelTasks from the prompts package public API
Accept an array of task objects and an options object
Each task has a title (string), a task function that returns a value or Promise, and an optional enabled flag
Support configurable concurrency via opts.concurrency (static number or dynamic function)
Implement opts.stopOnError to control whether execution halts on first failure
Return a results array preserving original task order with appropriate status for each task
Support an opts.onTaskComplete callback for progress tracking Interface Specification Result Object Schema: Each result in the returned array must have a status field with one of these values:
'success': Task completed successfully; includes a value field with the return value
'error': Task threw or rejected; includes an error field with the caught value
'pending': Task was never started (due to stopOnError halting execution)
'skipped': Task had enabled: false and was not executed onTaskComplete(index: number, result: ResultObject): void
Called immediately after each task completes (or is skipped)
index is the task's position in the original array
Errors thrown inside the callback should be swallowed (not cause overall rejection)
concurrency: Defaults to Infinity (no limit)
stopOnError: Defaults to true
When a task fails, no new tasks are started
Already-running tasks are allowed to complete normally
Tasks that were never started remain with status 'pending'
When dynamic concurrency returns zero or negative, pause scheduling until it returns a positive value
Implementations should tolerate unknown options in the options object
Environment Setup
Setup instructions and environment configuration for this problem.

Repository
https://github.com/bombshell-dev/clack
•
Commit: 5c65e3885f286ba0d38cb43e8d4833de80c17ec2
Quick Setup
Copy and paste this into your Linux or macOS terminal to set up the environment.

setup.sh

cat <<'EOSCRIPT' | bash
#!/bin/bash

# Clone repository and checkout commit
git clone https://github.com/bombshell-dev/clack clack-d5ebbc0b-4c95-4bbd-ab46-c0f3aebd9bbe --recurse-submodules
cd clack-d5ebbc0b-4c95-4bbd-ab46-c0f3aebd9bbe
git checkout 5c65e3885f286ba0d38cb43e8d4833de80c17ec2

# Create problem branch
git checkout -b shipd-problem/d5ebbc0b-4c95-4bbd-ab46-c0f3aebd9bbe-v19

# Create and apply test patch
cat > test.patch << 'EOF'
diff --git a/packages/prompts/test/parallel-tasks.test.ts b/packages/prompts/test/parallel-tasks.test.ts
new file mode 100644
index 0000000..a269d5a
--- /dev/null
+++ b/packages/prompts/test/parallel-tasks.test.ts
@@ -0,0 +1,366 @@
+import { afterEach, beforeEach, describe, expect, test, vi } from 'vitest';
+import * as prompts from '../src/index.js';
+import { MockWritable } from './test-utils.js';
+
+describe('parallelTasks', () => {
+	let output: MockWritable;
+
+	beforeEach(() => {
+		output = new MockWritable();
+		vi.useFakeTimers();
+	});
+
+	afterEach(() => {
+		vi.useRealTimers();
+		vi.restoreAllMocks();
+	});
+
+	describe('basic execution', () => {
+		test('executes all tasks and returns results in original order', async () => {
+			const tasks = [
+				{ title: 'Task 1', task: async () => 'result-1' },
+				{ title: 'Task 2', task: async () => 'result-2' },
+				{ title: 'Task 3', task: async () => 'result-3' },
+			];
+
+			const resultPromise = prompts.parallelTasks(tasks, { output });
+			await vi.runAllTimersAsync();
+			const results = await resultPromise;
+
+			expect(results).toHaveLength(3);
+			expect(results[0]).toMatchObject({ status: 'success', value: 'result-1' });
+			expect(results[1]).toMatchObject({ status: 'success', value: 'result-2' });
+			expect(results[2]).toMatchObject({ status: 'success', value: 'result-3' });
+		});
+
+		test('handles empty task array', async () => {
+			const resultPromise = prompts.parallelTasks([], { output });
+			await vi.runAllTimersAsync();
+			const results = await resultPromise;
+			expect(results).toEqual([]);
+		});
+	});
+
+	describe('concurrency control', () => {
+		test('defaults to unlimited concurrency (Infinity)', async () => {
+			const running: Set<number> = new Set();
+			let maxConcurrent = 0;
+			const tasks = Array.from({ length: 5 }, (_, i) => ({
+				title: `Task ${i}`,
+				task: async () => {
+					running.add(i);
+					maxConcurrent = Math.max(maxConcurrent, running.size);
+					await new Promise((r) => setTimeout(r, 50));
+					running.delete(i);
+					return i;
+				},
+			}));
+
+			const resultPromise = prompts.parallelTasks(tasks, { output });
+			await vi.runAllTimersAsync();
+			await resultPromise;
+
+			expect(maxConcurrent).toBe(5);
+		});
+
+		test('respects static concurrency limit', async () => {
+			const running: Set<number> = new Set();
+			let maxConcurrent = 0;
+			const tasks = Array.from({ length: 5 }, (_, i) => ({
+				title: `Task ${i}`,
+				task: async () => {
+					running.add(i);
+					maxConcurrent = Math.max(maxConcurrent, running.size);
+					await new Promise((r) => setTimeout(r, 50));
+					running.delete(i);
+					return i;
+				},
+			}));
+
+			const resultPromise = prompts.parallelTasks(tasks, { output, concurrency: 2 });
+			await vi.runAllTimersAsync();
+			await resultPromise;
+
+			expect(maxConcurrent).toBe(2);
+		});
+
+		test('supports dynamic concurrency function', async () => {
+			let limit = 1;
+			const tasks = [
+				{
+					title: 'Task 0',
+					task: async () => {
+						await new Promise((r) => setTimeout(r, 50));
+						limit = 3;
+						return 0;
+					},
+				},
+				{ title: 'Task 1', task: async () => 1 },
+				{ title: 'Task 2', task: async () => 2 },
+			];
+
+			const resultPromise = prompts.parallelTasks(tasks, {
+				output,
+				concurrency: () => limit,
+			});
+			await vi.runAllTimersAsync();
+			const results = await resultPromise;
+
+			expect(results.every((r) => r.status === 'success')).toBe(true);
+		});
+
+		test('handles negative concurrency by waiting for positive value', async () => {
+			let currentLimit = -1;
+			let taskStarted = false;
+			const tasks = [{
+				title: 'Delayed',
+				task: async () => {
+					taskStarted = true;
+					return 'done';
+				},
+			}];
+
+			setTimeout(() => { currentLimit = 1; }, 100);
+
+			const resultPromise = prompts.parallelTasks(tasks, {
+				output,
+				concurrency: () => currentLimit,
+			});
+
+			// Advance time partially - task should not have started yet
+			await vi.advanceTimersByTimeAsync(50);
+			expect(taskStarted).toBe(false);
+
+			// Advance past when limit becomes positive
+			await vi.runAllTimersAsync();
+			const results = await resultPromise;
+
+			expect(taskStarted).toBe(true);
+			expect(results[0]).toMatchObject({ status: 'success', value: 'done' });
+		});
+	});
+
+	describe('error handling', () => {
+		test('stops starting new tasks on error by default', async () => {
+			const started: number[] = [];
+			const tasks = [
+				{
+					title: 'Fails',
+					task: async () => {
+						started.push(0);
+						throw new Error('fail');
+					},
+				},
+				{
+					title: 'Never runs',
+					task: async () => {
+						started.push(1);
+						return 1;
+					},
+				},
+			];
+
+			const resultPromise = prompts.parallelTasks(tasks, { output, concurrency: 1 });
+			await vi.runAllTimersAsync();
+			const results = await resultPromise;
+
+			expect(started).toEqual([0]);
+			expect(results[0].status).toBe('error');
+			expect(results[1].status).toBe('pending');
+		});
+
+		test('continues all tasks when stopOnError is false', async () => {
+			const tasks = [
+				{ title: 'Fails', task: async () => { throw new Error('fail'); } },
+				{ title: 'Succeeds', task: async () => 'ok' },
+			];
+
+			const resultPromise = prompts.parallelTasks(tasks, { output, stopOnError: false });
+			await vi.runAllTimersAsync();
+			const results = await resultPromise;
+
+			expect(results[0].status).toBe('error');
+			expect(results[1]).toMatchObject({ status: 'success', value: 'ok' });
+		});
+
+		test('handles synchronous exceptions in task function', async () => {
+			const tasks = [{
+				title: 'Throws sync',
+				task: () => { throw new Error('sync error'); },
+			}];
+
+			const resultPromise = prompts.parallelTasks(tasks, { output });
+			await vi.runAllTimersAsync();
+			const results = await resultPromise;
+
+			expect(results[0].status).toBe('error');
+			expect((results[0] as any).error.message).toBe('sync error');
+		});
+	});
+
+	describe('enabled flag', () => {
+		test('skips disabled tasks without executing them', async () => {
+			const executed: string[] = [];
+			const tasks = [
+				{ title: 'Enabled', task: async () => { executed.push('a'); return 'a'; } },
+				{ title: 'Disabled', task: async () => { executed.push('b'); return 'b'; }, enabled: false },
+			];
+
+			const resultPromise = prompts.parallelTasks(tasks, { output });
+			await vi.runAllTimersAsync();
+			const results = await resultPromise;
+
+			expect(executed).toEqual(['a']);
+			expect(results[1]).toMatchObject({ status: 'skipped' });
+		});
+	});
+
+	describe('onTaskComplete callback', () => {
+		test('invokes callback as each task completes', async () => {
+			const completions: number[] = [];
+			const tasks = [
+				{
+					title: 'Slow',
+					task: async () => {
+						await new Promise((r) => setTimeout(r, 100));
+						return 'slow';
+					},
+				},
+				{
+					title: 'Fast',
+					task: async () => {
+						await new Promise((r) => setTimeout(r, 25));
+						return 'fast';
+					},
+				},
+			];
+
+			const resultPromise = prompts.parallelTasks(tasks, {
+				output,
+				onTaskComplete: (index) => completions.push(index),
+			});
+			await vi.runAllTimersAsync();
+			await resultPromise;
+
+			expect(completions).toEqual([1, 0]);
+		});
+
+		test('invokes callback for skipped tasks', async () => {
+			const completions: Array<{ index: number; status: string }> = [];
+			const tasks = [
+				{ title: 'Enabled', task: async () => 'a' },
+				{ title: 'Disabled', task: async () => 'b', enabled: false },
+			];
+
+			const resultPromise = prompts.parallelTasks(tasks, {
+				output,
+				onTaskComplete: (index, result) => {
+					completions.push({ index, status: result.status });
+				},
+			});
+			await vi.runAllTimersAsync();
+			await resultPromise;
+
+			expect(completions).toContainEqual({ index: 1, status: 'skipped' });
+		});
+	});
+
+	describe('edge cases', () => {
+		test('handles synchronous task functions', async () => {
+			const tasks = [{ title: 'Sync', task: () => 'sync-result' }];
+			const resultPromise = prompts.parallelTasks(tasks, { output });
+			await vi.runAllTimersAsync();
+			const results = await resultPromise;
+			expect(results[0]).toMatchObject({ status: 'success', value: 'sync-result' });
+		});
+
+		test('handles task rejecting with non-Error value', async () => {
+			const tasks = [{
+				title: 'Rejects string',
+				task: async () => { throw 'string-error'; },
+			}];
+
+			const resultPromise = prompts.parallelTasks(tasks, { output });
+			await vi.runAllTimersAsync();
+			const results = await resultPromise;
+
+			expect(results[0].status).toBe('error');
+			expect((results[0] as any).error).toBe('string-error');
+		});
+
+		test('handles onTaskComplete callback that throws', async () => {
+			const tasks = [
+				{ title: 'Task', task: async () => 'result' },
+			];
+
+			const resultPromise = prompts.parallelTasks(tasks, {
+				output,
+				onTaskComplete: () => { throw new Error('callback error'); },
+			});
+			await vi.runAllTimersAsync();
+
+			await expect(resultPromise).resolves.toBeDefined();
+		});
+
+		test('handles dynamic concurrency returning zero', async () => {
+			let allowStart = false;
+			let taskStarted = false;
+			const tasks = [{
+				title: 'Delayed',
+				task: async () => {
+					taskStarted = true;
+					return 'done';
+				},
+			}];
+
+			setTimeout(() => { allowStart = true; }, 100);
+
+			const resultPromise = prompts.parallelTasks(tasks, {
+				output,
+				concurrency: () => (allowStart ? 1 : 0),
+			});
+
+			// Advance time partially - task should not have started yet
+			await vi.advanceTimersByTimeAsync(50);
+			expect(taskStarted).toBe(false);
+
+			// Advance past when limit becomes positive
+			await vi.runAllTimersAsync();
+			const results = await resultPromise;
+
+			expect(taskStarted).toBe(true);
+			expect(results[0]).toMatchObject({ status: 'success', value: 'done' });
+		});
+
+		test('allows running tasks to complete after stopOnError triggers', async () => {
+			const completed: string[] = [];
+			const tasks = [
+				{
+					title: 'Fails fast',
+					task: async () => {
+						await new Promise((r) => setTimeout(r, 25));
+						throw new Error('fast fail');
+					},
+				},
+				{
+					title: 'Completes slow',
+					task: async () => {
+						await new Promise((r) => setTimeout(r, 100));
+						completed.push('slow');
+						return 'slow-done';
+					},
+				},
+				{ title: 'Never starts', task: async () => 'never' },
+			];
+
+			const resultPromise = prompts.parallelTasks(tasks, { output, concurrency: 2 });
+			await vi.runAllTimersAsync();
+			const results = await resultPromise;
+
+			expect(results[0].status).toBe('error');
+			expect(results[1]).toMatchObject({ status: 'success', value: 'slow-done' });
+			expect(results[2].status).toBe('pending');
+			expect(completed).toEqual(['slow']);
+		});
+	});
+});
diff --git a/test.sh b/test.sh
new file mode 100755
index 0000000..f0e3248
--- /dev/null
+++ b/test.sh
@@ -0,0 +1,18 @@
+#!/usr/bin/env bash
+set -euo pipefail
+cd "$(dirname "$0")"
+
+MODE="${1:-new}"
+
+if [ "$MODE" = "base" ]; then
+  pnpm build
+  cd packages/prompts
+  pnpm exec vitest run --exclude "**/parallel-tasks.test.ts"
+elif [ "$MODE" = "new" ]; then
+  pnpm build
+  cd packages/prompts
+  pnpm exec vitest run test/parallel-tasks.test.ts
+else
+  echo "Usage: $0 [base|new]" >&2
+  exit 1
+fi

EOF
git apply test.patch
rm test.patch

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM public.ecr.aws/x8v8d7g8/mars-base:latest
WORKDIR /app

ENV NODE_ENV=development
ENV NPM_CONFIG_PRODUCTION=false

COPY . .

RUN NODE_ENV=development pnpm install --production=false

CMD ["/bin/bash"]

EOF

# Commit changes
git add .
git commit -m "Add test environment for problem d5ebbc0b-4c95-4bbd-ab46-c0f3aebd9bbe-v19"
EOSCRIPT

# Navigate to project directory
cd clack

# Build and run Docker container (uncomment to use)
# docker build -t mars-problem .
# docker run -it mars-problem