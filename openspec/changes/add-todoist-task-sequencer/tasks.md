# Tasks

## 1. Project scaffolding

- [ ] 1.1 Initialize the Node 22 / TypeScript project: `package.json` with scripts `build`, `test`, `test:integration`, `authorize`, `deploy`, and `deploy:local`; strict `tsconfig.json`; `.gitignore`; `.nvmrc`. The `deploy:local` script runs `LSTK_CDK_CMD="npx cdk" lstk cdk deploy --all --require-approval never`. Verify that `npm install` and `npx tsc --noEmit` succeed.
- [ ] 1.2 Add the dependencies (`aws-cdk-lib`, `constructs`, `@aws-sdk/client-secrets-manager`, `vitest`, `esbuild`, `aws-cdk` (version 2.177.0 or newer), `tsx`; no `aws-cdk-local`) and create the directory layout from design D7. Verify that `npx vitest run` exits 0 with no tests, that `npx cdk --version` reports 2.177.0 or newer, and that `lstk --version` works.

## 2. Todoist app setup and API contract capture

- [ ] 2.1 Register a Todoist app in the App Management Console. Create a test token for it, and subscribe the app to `item:completed` and `item:deleted` with a temporary callback URL. Then complete a task in a scratch project and check whether a webhook arrives when only the test token was used (design D9). In the same session, check whether the console accepts `http://localhost:8765/callback` as the OAuth redirect URL. Verify by recording both answers in design.md D9. Whether task 6.1 is needed depends on these answers.
- [ ] 2.2 Using the scratch project, record real v1 responses for listing tasks (with pagination), listing sections, and updating a task with `due_string: "today"`. Also record real webhook payloads for `item:completed` and `item:deleted`. Save them, with personal data removed, under `test/fixtures/`. Verify that the fixtures exist and show where `project_id`, `parent_id`, `section_id`, `child_order`, and `due` are located. If v1 rejects `due_string` on update, record that and switch to the D3 fallback.

## 3. Sequencing logic (task-sequencing spec)

- [ ] 3.1 Implement `hasDatedTask(tasks)` and `selectNextTask(tasks, sections)` in `src/sequencer.ts`: unsectioned tasks first, then sections by `section_order`, then tasks by `child_order`; top-level tasks only. Verify with unit tests covering the scenarios "Unsectioned tasks come first", "Sections in order", "Subtasks are skipped", "Another task still has a date", "Project is empty", and "Recurring task completed" (a task that still has a `due`).
- [ ] 3.2 Implement `src/todoist-client.ts` (list tasks by project with cursor pagination, list sections by project, update a task's due date; configurable base URL; bearer token; typed errors for 401, 429, and 5xx). Verify with unit tests that stub `fetch` against the task 2.2 fixtures, including multi-page task lists and each error class.
- [ ] 3.3 Implement the sequencing service (`runSequencing(projectId, client)`), which composes 3.1 and 3.2 and returns `{ promoted: taskId | null }`. Verify with unit tests for "Current task completed, next task promoted", the duplicate-delivery case (the second run is a no-op), and the "Todoist API unavailable" error propagation.

## 4. Webhook intake (todoist-webhook-intake spec)

- [ ] 4.1 Implement `src/signature.ts` (Base64 HMAC-SHA256 of the raw body, compared with `timingSafeEqual`). Verify with unit tests for the "Valid signature", "Missing signature", and "Tampered body" scenarios, plus a Base64-encoded Function URL body.
- [ ] 4.2 Implement `src/config.ts`: read `TODOIST_PROJECT_ID`, `TODOIST_API_BASE_URL`, and `SECRET_ARN`, and fetch the secret once per container from Secrets Manager. Verify with a unit test that a mocked Secrets Manager client is called only once across two invocations.
- [ ] 4.3 Implement `src/handler.ts`: 405 for non-POST; 401 for a bad signature; 400 for a malformed payload; 200 no-op for other event types or projects; run sequencing for `item:completed` and `item:deleted` in the configured project; 200 on success and 502 on Todoist failure. Use structured JSON logs that never include secrets, tokens, or signatures. Verify with unit tests covering every webhook-intake scenario except the concurrency ones (Lambda enforces those; see 5.1 and 7.3), plus a log-capture test asserting that no secret values appear.

## 5. Infrastructure (CDK)

- [ ] 5.1 Implement `infra/stack.ts` with:
  - a Secrets Manager secret with placeholder JSON;
  - a `NodejsFunction` (nodejs22.x, arm64, 15 s timeout, `reservedConcurrency` prop defaulting to 1) with read access to the secret;
  - a Function URL (`authType: NONE`);
  - env vars from the CDK context values `projectId` and `todoistApiBaseUrl`;
  - `CfnOutput`s for `WebhookUrl` and `SecretArn`.

  Verify with a CDK assertions unit test (`Template.fromStack`) checking the function, `ReservedConcurrentExecutions: 1`, the URL, the secret grant, and the outputs.
- [ ] 5.2 Add the AWS Budgets monthly cost budget (design D10): `budgetLimitUsd` prop defaulting to 5; email notifications at 80% and 100% of actual spend, sent to the `budgetEmail` context value; skipped with a synth warning when `budgetEmail` isn't set. Verify with a CDK assertions test for the `AWS::Budgets::Budget` resource and its two notifications, and a test that it's omitted when no email is given.
- [ ] 5.3 Wire `infra/app.ts` and `cdk.json`. Verify that `npx cdk synth -c projectId=test -c budgetEmail=test@example.com` produces a template without errors.

## 6. OAuth fallback helper (only if needed)

- [ ] 6.1 Only if task 2.1 found that the test token does **not** trigger webhooks: implement `scripts/authorize.ts` (design D9). It builds the authorize URL with a random `state`, listens for the callback on `localhost:8765`, exchanges the code, and writes `{clientSecret, accessToken}` to the secret ARN from stack outputs. Verify with a unit test of the URL and `state` handling and of the token-exchange request shape (with stubbed `fetch`), and by a manual run against the real Todoist app. If the test token works, mark this task done, remove the `authorize` npm script, and note the outcome in design.md D9.

## 7. LocalStack integration tests

- [ ] 7.1 Implement `test/mock-todoist/` as an in-process HTTP server for vitest. It holds tasks and sections in memory and serves the v1 endpoints for list tasks, list sections, and update task, resolving `due_string: "today"` to a configured fixed date. It exposes an exported API for setting and reading state and the recorded calls (no HTTP control endpoints), a switchable failure mode (forced 500 or 401), and a configurable response delay for concurrency tests. Verify with a unit test that exercises it.
- [ ] 7.2 Add the project-local `.lstk/config.toml` with a `test` env profile setting `LAMBDA_DOCKER_FLAGS="--add-host=host.docker.internal:host-gateway"`, and add an `npm run localstack:start` script (`lstk --non-interactive start`). Verify that `lstk start` reports the emulator ready, and that `lstk aws lambda list-functions` succeeds.
- [ ] 7.3 Add an integration global setup that:
  - checks that LocalStack is running (`lstk --non-interactive status`) and fails with a clear message if not;
  - starts the mock on port 8787;
  - runs `lstk cdk bootstrap` and `lstk cdk deploy` (through `LSTK_CDK_CMD="npx cdk"`) with `projectId` set to the test project and `todoistApiBaseUrl=http://host.docker.internal:8787/api/v1`;
  - seeds the secret;
  - reads the `WebhookUrl` output;
  - sends one signed probe event to confirm the Lambda can reach the mock, and fails with a clear message about `host.docker.internal` if it can't.

  Verify that `npm run test:integration` gets through setup and prints the Function URL.
- [ ] 7.4 Write integration tests against the deployed Function URL for these scenarios:
  - completion promotes the next task;
  - deletion promotes the next task;
  - another dated task blocks promotion;
  - `item:updated` is ignored;
  - the other project is ignored;
  - a bad signature returns 401 with zero mock calls;
  - a duplicate delivery is a no-op;
  - a mock 500 returns 5xx.

  Verify that `npm run test:integration` passes with all scenarios green on LocalStack.
- [ ] 7.5 Write the integration test "Concurrent deliveries": with the mock's response delay set to about 2 s, send two signed deliveries at once and check that one gets 200, the other gets 429, and exactly one task ends up dated. Verify that it passes on LocalStack. If LocalStack doesn't enforce reserved concurrency for Function URLs, mark the test as skipped with a comment explaining why, keep the 5.1 template assertion as the coverage, and record the gap in design.md D8.

## 8. Documentation and end-to-end check

- [ ] 8.1 Write `README.md` covering:
  - architecture overview;
  - cost-protection rationale (reserved concurrency, the ~$2/day ceiling, the budget alert);
  - prerequisites (Node 22, Docker, `lstk`, a LocalStack account);
  - Todoist app registration and webhook subscription (`item:completed`, `item:deleted`);
  - the test-token setup (or `npm run authorize` if 6.1 was needed);
  - AWS deploy with `projectId` and `budgetEmail`;
  - running unit tests, and LocalStack tests via `lstk login` / `lstk start` (or `LOCALSTACK_AUTH_TOKEN` with `--non-interactive` in CI);
  - bootstrapping the first dated task;
  - recovering from a stalled chain;
  - the concurrency-quota caveat;
  - rollback.

  Verify by following the LocalStack section from a clean checkout and confirming the commands run as written.
- [ ] 8.2 Final integration check: run `npm test`, `npm run test:integration`, and `npx cdk synth -c projectId=test`, all green. Then run `openspec validate add-todoist-task-sequencer --strict` and confirm it passes.
