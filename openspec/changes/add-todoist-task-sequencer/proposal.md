> [!NOTE]
> **Annotations present.** This file contains reviewer feedback.
> `==highlights==` flag text for discussion. `%%comments%%` are inline notes (hidden in preview, visible in source).
> `~~deletions~~` suggest removal. `> [!EDIT]` blocks are change requests.
> These markers are intentional — do not remove or "clean up" without asking the reviewer.

# Proposal

## Why

I want to work through one Todoist project in a fixed order, one task at a time, without the whole project crowding my "Today" view. Today I have to pick the next task and give it a date by hand every time I finish one. That step is easy to forget, so the sequence stalls. An automated trigger keeps exactly one task from the project on "Today" indefinitely.

## What Changes

- Add a webhook-driven trigger, deployed on AWS serverless infrastructure and written in TypeScript. It reacts to Todoist task completions and deletions in one configured project.
- When a task in the project is completed or deleted and no open task in the project still has a date, the trigger gives the topmost open task (in manual order) a due date of **today**. The task then appears in Todoist's "Today" view.
- The trigger ignores manual date edits. If the user reschedules, adds a date, or removes a date by hand, it does nothing.
- The trigger checks every incoming webhook against the Todoist app's client secret and rejects anything it can't verify.
- The trigger handles one delivery at a time. A delivery that arrives while another is still running gets a 429, and Todoist retries it later. This also limits how much it costs if someone spams the public endpoint.
- Define the infrastructure as code (AWS CDK in TypeScript): a Lambda function exposed through a public HTTPS endpoint, with secrets kept in AWS Secrets Manager.
- Run all integration tests against LocalStack using the user's LocalStack Auth Token, with a mock Todoist API. Tests never touch the real Todoist account.
- Document the one-time Todoist app registration and OAuth authorization. Todoist only delivers webhooks for users who have authorized the app.

## Capabilities

### New Capabilities

- `task-sequencing`: Rules for picking the single "next" task in the configured project and giving it today's date. Covers what triggers it, how the next task is chosen, when it does nothing, and idempotency.
- `todoist-webhook-intake`: How the trigger accepts Todoist webhook deliveries. Covers signature verification, event and project filtering, one-at-a-time processing, and response codes.

### Modified Capabilities

None. This is a new project with no existing specs.

## Impact

- **New codebase**: TypeScript Node.js project (Lambda handler, Todoist API client, CDK app, unit and integration tests).
- **External systems**: Todoist API v1 (read tasks, update a task's due date) and Todoist webhooks (`item:completed`, `item:deleted`). This needs a registered Todoist app and a one-time OAuth authorization.
- **AWS resources**: one Lambda function with a Function URL and reserved concurrency of 1, which caps spam cost at about $2/day; one Secrets Manager secret (client secret and access token); CloudWatch Logs; and an AWS Budgets cost alert.
- **Tooling**: AWS CDK, the `lstk` LocalStack CLI (`lstk start`, `lstk cdk`), Docker, a LocalStack Auth Token (`lstk login` or `LOCALSTACK_AUTH_TOKEN`), and a test runner.
