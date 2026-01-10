Repository
https://github.com/bombshell-dev/clack
Commit: 5c65e3885f286ba0d38cb43e8d4833de80c17ec2

Title
Add parallel task execution for concurrent CLI operations

Implement a `parallelTasks()` function that enables executing multiple CLI tasks concurrently with configurable concurrency limits. This feature allows users to run independent operations in parallel while maintaining control over resource usage and error handling behavior.
- Export `parallelTasks` from the prompts package public API
- Accept an array of task objects and an options object
- Each task has a `title` (string), a `task` function that returns a value or Promise, and an optional `enabled` flag
- Support configurable concurrency via `opts.concurrency` (static number or dynamic function)
- Implement `opts.stopOnError` to control whether execution halts on first failure
- Return a results array preserving original task order with appropriate status for each task
- Support an `opts.onTaskComplete` callback for progress tracking
Interface Specification
Result Object Schema:
Each result in the returned array must have a `status` field with one of these values:
- `'success'`: Task completed successfully; includes a `value` field with the return value
- `'error'`: Task threw or rejected; includes an `error` field with the caught value
- `'pending'`: Task was never started (due to stopOnError halting execution)
- `'skipped'`: Task had `enabled: false` and was not executed
`onTaskComplete(index: number, result: ResultObject): void`
- Called immediately after each task completes (or is skipped)
- `index` is the task's position in the original array
- Errors thrown inside the callback should be swallowed (not cause overall rejection)
- `concurrency`: Defaults to `Infinity` (no limit)
- `stopOnError`: Defaults to `true`
  - When a task fails, no new tasks are started
  - Already-running tasks are allowed to complete normally
  - Tasks that were never started remain with status `'pending'`
- When dynamic concurrency returns zero or negative, pause scheduling until it returns a positive value
- Implementations should tolerate unknown options in the options object

