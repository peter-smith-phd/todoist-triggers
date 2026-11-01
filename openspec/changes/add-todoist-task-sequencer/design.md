> [!NOTE]
> **Annotations present.** This file contains reviewer feedback.
> `==highlights==` flag text for discussion. `%%comments%%` are inline notes (hidden in preview, visible in source).
> `~~deletions~~` suggest removal. `> [!EDIT]` blocks are change requests.
> These markers are intentional — do not remove or "clean up" without asking the reviewer.

# Design

## Context

This is a greenfield repository: it contains only the OpenSpec scaffolding. See proposal.md (Why) for the motivation and the specs for the required behavior:

- `specs/task-sequencing/spec.md`
- `specs/todoist-webhook-intake/spec.md`

Constraints that shape the design:

- **Todoist webhooks need a registered app plus OAuth.** Todoist only delivers webhooks for users who have authorized the app through OAuth. A personal API token doesn't trigger them. Each delivery is signed with the app's client secret (`X-Todoist-Hmac-SHA256`).
- **Todoist webhooks cover the whole account.** Every `item:completed` and `item:deleted` in any of the user's projects reaches the endpoint. Filtering by project happens in the handler.
- **Todoist API v1** (`https://api.todoist.com/api/v1`) is the current unified API. It replaced REST v2 and Sync v9. Tasks expose `project_id`, `section_id`, `parent_id`, `child_order`, `due`, and `is_completed`. List endpoints are paginated with a `cursor`.
- **AWS serverless with TypeScript.** Every integration test runs on LocalStack, managed with the official `lstk` CLI. `lstk` runs its own container and can't be combined with a Docker Compose LocalStack setup.
- **The user's decisions:** use webhooks, use AWS CDK, act only on completions and deletions, ignore manual date edits, put cost-risk reduction ahead of a strict request-rate limit, and run the Todoist mock on the host.

## Goals / Non-Goals

**Goals:**
- Keep the handler stateless. Each qualifying event re-reads the project from Todoist and checks the rule "no open task is dated → date the top task". There's no database to drift out of sync with Todoist.
- Make it cheap to run, and put a hard cap on what a spammer can cost: one Lambda, no always-on resources, no paid throttling layer.
- Make the deployment fully reproducible on LocalStack with the same CDK app used for AWS.

**Non-Goals:**
- Supporting several projects or several users. One project ID and one OAuth token per deployment.
- A scheduled reconcile or polling fallback. It would undo the user's manual date removals, which the spec says to ignore.
- A hosted OAuth callback endpoint. Authorization is a one-time local step.
- A strict requests-per-second limit (API Gateway throttling), custom domains, WAF, or multi-region deployment. See D1 for why concurrency caps cost better than a rate limit.

## Decisions

### D1: Lambda Function URL with reserved concurrency of 1, not a throttled API Gateway
The webhook is one unauthenticated POST route that we authenticate ourselves with HMAC. A Function URL (`authType: NONE`) gives an HTTPS endpoint with no extra service, and LocalStack supports it.

The main risk is cost: anyone can POST to the endpoint. Each forged request is rejected by the HMAC check in about 10 ms, but it's still a billed invocation (about $0.22 per million at arm64/128 MB). Reserved concurrency of 1 (D5) caps throughput at roughly 100 rejected requests per second. Lambda doesn't bill requests it rejects with a 429, so the **worst case is about $1.90/day** however hard the spammer pushes. An AWS Budgets alert (D10) reports anything unusual well before that adds up.

Estimated cost per day by spam rate:

| Setup | 10 req/s | 100 req/s | 1,000 req/s | Ceiling |
|---|---|---|---|---|
| Function URL, no concurrency cap | $0.19 | $1.90 | $19 | ~$1,900/day (account concurrency quota) |
| **Function URL + reserved concurrency 1 (chosen)** | $0.19 | ~$1.90 | ~$1.90 | **~$1.90/day** |
| HTTP API throttled at 1 req/s, throttled requests free | ~$0.09 | ~$0.09 | ~$0.09 | ~$0.11/day |
| HTTP API throttled at 1 req/s, throttled requests billed | $0.86 | $8.64 | $86 | ~$860/day (10k req/s account limit) |

*Alternatives:*
- **API Gateway HTTP API with stage throttling.** It gives a true 1 req/s limit, but it costs $1.00 per million requests. AWS doesn't document whether throttled requests are billed. If they are, this setup costs more under spam than the chosen one, not less. LocalStack also likely doesn't enforce the throttle, so we couldn't test it.
- **WAF rate-based rule.** It can't attach to a Function URL or an HTTP API, and it costs at least $5/month.

### D2: Stateless rule check instead of tracking the "current" task
On every qualifying event, the handler:
1. Lists open tasks in the project, following every page.
2. Stops if any open task has a `due`.
3. Otherwise lists the project's sections and sorts top-level tasks by (unsectioned first, then `section_order`, then `child_order`).
4. Sets the first task's due date to today.

This makes duplicate and out-of-order deliveries harmless and handles completions, deletions, and recurring tasks the same way.
*Alternative:* Storing the "current task ID" in DynamoDB. That adds state that drifts whenever the user edits the project, and it isn't needed.

### D3: Let Todoist resolve "today"
The handler sets the date with `due_string: "today"` (and `due_lang: "en"`). Todoist resolves it in the user's account timezone and creates a date-only due date. This avoids timezone handling in Lambda, which runs in UTC.
*Fallback (only if v1 rejects `due_string` on update):* Read the user's timezone from the user endpoint and send `due_date: YYYY-MM-DD`, computed with `Intl.DateTimeFormat`.

### D4: Credentials in one Secrets Manager secret
The secret is JSON: `{ "clientSecret": "...", "accessToken": "..." }`. CDK creates it with placeholder values, and the user fills it in once. The Lambda reads it on cold start and caches it in module scope. The project ID isn't sensitive, so it goes in a plain environment variable (`TODOIST_PROJECT_ID`), set from a CDK context value.
*Alternative:* SSM SecureString parameters. They'd work too, but they'd mean two parameters, and Secrets Manager is simpler to seed on LocalStack.

### D5: Process synchronously, one delivery at a time, and return 5xx on failure
The handler runs at most three or four Todoist calls, well inside Todoist's webhook timeout. Returning `5xx` on failure uses Todoist's built-in retries. No SQS queue or dead-letter queue is needed. Lambda timeout: 15 s.

Reserved concurrency is set to 1, which serves two purposes:
- It removes the read-then-write race in "two completions in quick succession".
- It caps spam cost (D1).

Lambda rejects a concurrent request with a 429, and Todoist retries it. This is the "one at a time" requirement in the webhook-intake spec.

**Reserved concurrency is free.** It only sets aside part of the account's concurrency quota. Provisioned concurrency is the paid feature, and we don't use it. The catch is the quota: AWS requires at least 100 unreserved concurrent executions to remain, and some new accounts start with a limit of 10, where even reserving 1 fails. The value is a CDK prop (`reservedConcurrency`, default `1`) so it can be changed. On such an account, ask AWS to raise the quota. Don't disable the cap, because it is the cost control.

### D6: Thin hand-written Todoist client using `fetch`
The client needs only three calls: list tasks, list sections, and update a task. It uses Node's built-in `fetch`, typed with local interfaces, so there's no SDK dependency or bundle weight, and the version is pinned to the API we test against. The base URL is configurable (`TODOIST_API_BASE_URL`) so integration tests can point it at the mock.
*Alternative:* The official `@doist/todoist-api-typescript` SDK. It's an option if it fully supports v1 when we implement, but it would make the base URL harder to swap.

### D7: Code layout and toolchain
```
src/
  handler.ts          Function URL entry point: method check, signature, parse, filter, respond
  signature.ts        HMAC-SHA256 base64 verify (crypto.timingSafeEqual)
  sequencer.ts        pure functions: hasDatedTask(), selectNextTask(tasks, sections)
  todoist-client.ts   fetch-based v1 client with pagination
  config.ts           env + cached Secrets Manager read
infra/
  app.ts, stack.ts    CDK app: NodejsFunction (esbuild, nodejs22.x, arm64), Function URL,
                      reserved concurrency, Secret, Budget, outputs
scripts/
  authorize.ts        fallback OAuth code-flow helper (only if the test token doesn't work, see D9)
test/
  unit/               vitest, no network
  integration/        vitest against LocalStack; global setup deploys the stack
  mock-todoist/       in-process Node HTTP server emulating the v1 endpoints used
.lstk/config.toml     project-local lstk profile for integration tests
```
Toolchain: Node 22, TypeScript (strict), vitest, esbuild (through CDK `NodejsFunction`), `aws-cdk-lib`, `aws-cdk` (CLI 2.177.0 or newer, as a dev dependency), and `lstk`. We don't use `aws-cdk-local`: `lstk cdk` replaces `cdklocal`. npm scripts run it with `LSTK_CDK_CMD="npx cdk"`, so no global CDK install is needed.

### D8: Integration testing on lstk-managed LocalStack with the Todoist mock on the host
```
 +--------------------------------+        +------------------------------+
 | host                           |        | lstk-managed LocalStack      |
 |  vitest                        |  POST  |  Function URL                |
 |   +-- mock Todoist (in-process)| -----> |    |                         |
 |   |    on :8787                |        |    v                         |
 |   |                            | <----- |  Lambda container            |
 |   +-- asserts mock state       |  HTTP  |   TODOIST_API_BASE_URL =     |
 |       directly                 |        |   http://host.docker.internal|
 +--------------------------------+        |   :8787/api/v1               |
                                           +------------------------------+
```
- **Starting LocalStack:** `lstk start` (`lstk --non-interactive start` in CI). Auth comes from `lstk login`, or from `LOCALSTACK_AUTH_TOKEN` in CI. The integration setup checks that LocalStack is running (`lstk status`) and gives a clear error if it isn't. It doesn't manage the LocalStack container itself.
- **Reaching the host:** Lambda containers reach the host through `host.docker.internal`. Docker Desktop (macOS) provides this automatically. For Linux and CI, the project-local `.lstk/config.toml` adds a profile:
  ```toml
  [env.test]
  LAMBDA_DOCKER_FLAGS = "--add-host=host.docker.internal:host-gateway"
  ```
- **Test flow:** the vitest global setup does the following.
  1. Starts the mock Todoist server in-process on a fixed port.
  2. Deploys the stack with `lstk cdk bootstrap`, then `lstk cdk deploy --all --require-approval never -c projectId=<test> -c todoistApiBaseUrl=http://host.docker.internal:8787/api/v1`.
  3. Seeds the secret with a known test client secret and token (`lstk aws secretsmanager put-secret-value` or the AWS SDK pointed at LocalStack).
  4. For each scenario: sets the mock's state directly in memory, sends a correctly or incorrectly signed webhook to the Function URL, and checks the mock's recorded state and calls.
- **Deterministic dates:** the mock resolves `due_string: "today"` to a fixed configured date.
- **What each test layer covers:** integration tests cover every spec scenario that crosses the HTTP and AWS boundary, including concurrent deliveries returning 429. The ordering edge cases are covered by unit tests on `selectNextTask`.

*Alternative:* deploying the mock into LocalStack as a second Lambda, with its state in DynamoDB. It works the same on every OS, but it adds a test-only stack and makes the mock's state awkward to manage.

### D9: One-time OAuth: App Console test token first, local script as fallback
**Main route.** The Todoist App Management Console can create a test token for the app owner's own account. If that token counts as authorizing the app (so webhooks fire), setup is: copy the client secret and the test token into the secret once.

**Fallback.** If the test token doesn't trigger webhooks, `npm run authorize` runs the standard authorization-code flow. Todoist's server never contacts the local machine. The redirect is an HTTP 302 that the user's **browser** follows to `localhost`:
```
 browser                      Todoist                      your laptop
   | -- GET /oauth/authorize --> |                              |
   | <-- login + "Allow" page -- |                              |
   | -- user clicks Allow -----> |                              |
   | <-- 302 Location: http://localhost:8765/callback?code=..  |
   | -------------------------- GET /callback?code=... -------> |  (script)
   |                              | <-- POST /oauth/access_token |
   |                              | --- { access_token } ------> |
```
The script checks `state`, exchanges the code (an outbound call to Todoist), and writes `{clientSecret, accessToken}` to the deployed secret. This needs the App Console to accept `http://localhost:8765/callback` as the redirect URL. Task 6.1 checks that, along with the test token.

### D10: AWS Budgets alert as a cost backstop
The stack creates an AWS Budgets monthly cost budget, with a default limit of $5 (CDK prop). It emails the address given as a CDK context value when actual spend reaches 80% and 100%. The first two budgets in an account are free. This alert, together with D1's cost ceiling, means unexpected costs get noticed quickly.

## Risks / Trade-offs

- **[Todoist stops retrying a delivery]** → The chain stalls with nothing dated. *Mitigation:* Failures are logged in CloudWatch. The README explains how to restart: give the top task today's date by hand. No reconcile is added, because it would conflict with "ignore manual date removal".
- **[Webhook payload or API shape differs from assumptions]** (for example, how `item:deleted` carries `project_id` in v1) → *Mitigation:* An early task captures real sample payloads from a test project and pins them as unit-test fixtures before the handler logic is finalized.
- **[Account concurrency quota too low to reserve 1]** → Deployment fails. *Mitigation:* Request a Lambda concurrency quota increase. The cap shouldn't be removed, because it's the cost control (D5).
- **[Bursts of my own completions are throttled]** → Webhooks cover the whole account, so finishing several tasks within a second gives some of them a 429. *Mitigation:* Todoist retries them, and events from other projects are ignored anyway.
- **[Spam still costs something]** → Up to about $1.90/day while an attack lasts. *Mitigation:* Budget alert (D10). If it's ever needed, the Function URL can be rotated by redeploying the stack and updating the App Console callback URL.
- **[The OAuth token is revoked or the app is deauthorized]** → Every request returns 5xx, and webhooks also stop. *Mitigation:* The 401 from Todoist is logged clearly. Recovery is to issue a new test token or rerun `npm run authorize`.
- **[`host.docker.internal` isn't reachable from Lambda containers]** (Linux without the lstk profile) → *Mitigation:* The `.lstk/config.toml` profile, and a setup check that gives a clear error when the mock isn't reachable. The stack code is the same for AWS and LocalStack, apart from `TODOIST_API_BASE_URL`.

## Migration Plan

1. Register a Todoist app in the App Management Console. Note the client ID and client secret.
2. Deploy with `npx cdk deploy -c projectId=<id> -c budgetEmail=<email>`. The outputs are the Function URL and the secret ARN.
3. Set the app's webhook callback URL to the Function URL. Subscribe to `item:completed` and `item:deleted`, then activate webhooks.
4. Store `{clientSecret, accessToken}` in the secret, using the App Console test token (or `npm run authorize` as the fallback).
5. Bootstrap: make sure exactly one task (the top one) has today's date, or none does and you complete or delete a task.

**Rollback:** Deactivate the webhook in the App Console, which stops the trigger right away. Then run `cdk destroy`. Tasks in Todoist aren't affected.

## Open Questions

- Does Todoist v1 accept `due_string` on the task update endpoint as REST v2 did? If not, use the D3 fallback. Either way the behavior is the same.
- What is the exact retry schedule for failed Todoist webhook deliveries? This is informational only and goes in the README.
