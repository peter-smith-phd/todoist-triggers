# Spec Delta

## Purpose

A public HTTPS endpoint that receives Todoist webhook deliveries. It accepts only authentic deliveries and passes qualifying events to task sequencing.

## ADDED Requirements

### Requirement: Public HTTPS webhook endpoint
The system SHALL expose one HTTPS endpoint that accepts `POST` requests carrying a Todoist webhook JSON payload. The endpoint URL SHALL be published as a deployment output so the user can register it as the Todoist app's webhook callback URL.

#### Scenario: Endpoint published after deployment
- **WHEN** the stack is deployed
- **THEN** the deployment outputs include the webhook endpoint URL

#### Scenario: Non-POST request
- **WHEN** a request uses any method other than `POST`
- **THEN** the system responds with `405` and takes no action

### Requirement: Webhook signature verification
The system SHALL verify each delivery's `X-Todoist-Hmac-SHA256` header. The expected value is the Base64-encoded HMAC-SHA256 of the raw request body, keyed with the Todoist app's client secret, compared in constant time. Requests with a missing or mismatched signature MUST be rejected with `401`, with no processing and no outbound Todoist API calls.

#### Scenario: Valid signature
- **WHEN** a delivery has a signature that matches its raw body
- **THEN** the system processes the event

#### Scenario: Missing signature
- **WHEN** a delivery has no `X-Todoist-Hmac-SHA256` header
- **THEN** the system responds with `401` and makes no Todoist API calls

#### Scenario: Tampered body
- **WHEN** a delivery's body has been changed after signing
- **THEN** the system responds with `401` and makes no Todoist API calls

### Requirement: Event filtering
The system SHALL act only on `item:completed` and `item:deleted` events whose task belongs to the configured project. It SHALL acknowledge every other authentic delivery with `200` and take no action, so Todoist does not retry it.

#### Scenario: Qualifying event
- **WHEN** an authentic `item:completed` or `item:deleted` delivery names the configured project
- **THEN** the system runs the task-sequencing rule

#### Scenario: Non-qualifying event type
- **WHEN** an authentic delivery has any other event name (for example `item:updated` or `item:added`)
- **THEN** the system responds with `200` and makes no Todoist API calls

#### Scenario: Other project
- **WHEN** an authentic `item:completed` delivery names a different project
- **THEN** the system responds with `200` and makes no Todoist API calls

#### Scenario: Malformed payload
- **WHEN** an authentic delivery has a body that is not valid JSON or has no event name or project
- **THEN** the system responds with `400` and makes no Todoist API calls

### Requirement: Response codes reflect processing outcome
The system SHALL respond with `200` after an event is processed (whether or not a task was promoted). It SHALL respond with a `5xx` status when the sequencing rule fails, so Todoist retries the delivery.

#### Scenario: Processing succeeds
- **WHEN** a qualifying event is processed without error
- **THEN** the system responds with `200`

#### Scenario: Processing fails
- **WHEN** the sequencing rule fails because of a Todoist API error
- **THEN** the system responds with a `5xx` status

### Requirement: Deliveries are processed one at a time
The system SHALL process at most one webhook delivery at a time. A delivery that arrives while another is in progress MUST be rejected with `429` and MUST NOT make any Todoist API calls, so Todoist retries it later. This also caps the cost of a flood of requests, whether genuine or forged.

#### Scenario: Concurrent deliveries
- **WHEN** two deliveries arrive at the same moment
- **THEN** one is processed and the other gets `429` with no Todoist API calls

#### Scenario: Retried delivery succeeds
- **WHEN** a delivery that was rejected with `429` is retried after the earlier delivery finishes
- **THEN** the retry is processed normally

#### Scenario: Request flood
- **WHEN** many requests, genuine or forged, arrive at the same time
- **THEN** no more than one is processed at once, and the rest get `429`

### Requirement: Secrets are not exposed
The system MUST NOT write the Todoist client secret, the OAuth access token, or webhook signatures to logs or HTTP responses. It SHALL keep them in a managed secret store, not in source code or plain environment variables.

#### Scenario: Logs omit credentials
- **WHEN** any request is processed, whether it succeeds or fails
- **THEN** the emitted logs contain no client secret, no access token, and no signature value
