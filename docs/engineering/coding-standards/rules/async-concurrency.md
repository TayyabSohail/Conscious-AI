# Async Concurrency Standards

Run independent async operations concurrently with `Promise.all`. Never use `await`
inside a `for` loop. This is a **critical performance rule** — serializing async calls
can cause 10x slowdowns.

## Rules

### ALWAYS use `Promise.all` for parallel operations

- **When you need multiple independent results**, use `Promise.all`.
- **When one operation fails**, `Promise.all` rejects immediately (fail-fast).
- **If you need partial failures**, use `Promise.allSettled`.
- **CRITICAL**: Identify all independent async calls and wrap them in `Promise.all` —
  don't wait for one before starting the next unless there's a data dependency.

```ts
// ✅ Good: parallel operations
async function getUserWithDashboard(userId: string) {
  const [user, dashboard] = await Promise.all([
    fetchUser(userId),
    fetchDashboard(userId),
  ]);
  return { user, dashboard };
}

// ✅ Good: allSettled for partial failures
async function notifyMultipleUsers(userIds: string[]) {
  const results = await Promise.allSettled(
    userIds.map(id => sendEmail(id))
  );
  const successful = results.filter(r => r.status === 'fulfilled').length;
  return successful;
}
```

### Never `await` inside a `for` loop

- **This serializes operations** and is slow when they could run in parallel.
- **Rewrite to use `Promise.all` or `.map()` + `Promise.all`**.

```ts
// ❌ NEVER: sequential (10x slower)
async function createMultipleProjects(projects: CreateProjectInput[]) {
  const created = [];
  for (const project of projects) {
    const result = await createProject(project); // waits 1 second each = 10 seconds total
    created.push(result);
  }
  return created;
}

// ✅ ALWAYS: parallel with Promise.all
async function createMultipleProjects(projects: CreateProjectInput[]) {
  return Promise.all(projects.map(project => createProject(project))); // 1 second total
}

// ✅ Good: with allSettled (allow partial failures)
async function createMultipleProjects(projects: CreateProjectInput[]) {
  const results = await Promise.allSettled(
    projects.map(project => createProject(project))
  );
  const successful = results.filter(r => r.status === 'fulfilled');
  return successful.map(r => r.value);
}
```

### `for...of` with dependent operations is OK

- **If operation N+1 depends on the result of operation N**, using `await` in the loop
  is correct and necessary.

```ts
// ✅ Good: dependent operations (correct to await in loop)
async function processProjectWithTasks(projectId: string) {
  const project = await fetchProject(projectId);
  const tasks = await fetchTasks(projectId); // depends on first operation
  const topTask = await enrichTask(tasks[0]); // depends on previous step
  return { project, tasks, topTask };
}
```
