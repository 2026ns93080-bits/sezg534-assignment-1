# Feature Specification: Tiered Rate Limiter

**Feature Branch**: `tiered-rate-limiter`

**Created**: 2026-10-30

**Status**: Ready for Planning

**Input**: Build a rate-limiting microservice that lets a calling system check whether a client request may proceed. Each request identifies a client and its tier. The service applies configurable token-bucket limits for free (10 requests per minute), pro (100 requests per minute), and enterprise (1000 requests per minute); tracks each client's bucket independently; consumes a token for each allowed request; refills over time up to bucket capacity; reports remaining capacity for allowed requests and retry time for rejected requests; and rejects malformed, incomplete, or unknown-tier requests without consuming tokens. State is in memory only. Authentication, persistence, distributed or multi-instance operation, dashboards, and user interfaces are out of scope.

## Clarifications

### Session 2026-10-30

- Q: How should the service respond when `tier` is missing, null, or not a string, as well as when it names an unsupported tier? → A: Return HTTP 400 with `{"error":"tier must be one of: free, pro, enterprise"}` for every missing, null, non-string, or unsupported `tier`.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Check whether a client request may proceed (Priority: P1)

As a calling system, I want to submit a client's identifier and tier and receive an allow or reject decision, so that I can enforce the client's rate limit before processing its request.

**Why this priority**: This is the service's primary purpose and delivers value on its own.

**Independent Test**: Submit requests for a configured client and verify the service allows requests while tokens remain, then rejects requests when the bucket is empty and reports the retry delay.

**Acceptance Scenarios**:

1. **Given** a new client with a configured bucket containing two tokens, **When** the client makes two requests, **Then** both requests are allowed and the returned remaining counts are 1 and 0.
2. **Given** that client's bucket is empty and refills at two tokens per minute, **When** it makes another request immediately, **Then** it is rejected and told to retry in 30 seconds.
3. **Given** the same empty bucket, **When** 30 seconds elapse and the client makes a request, **Then** the request is allowed and the returned remaining count is 1.

### User Story 2 - Keep client limits independent and tier limits configurable (Priority: P2)

As a service operator, I want tier limits to be configurable and each client to have independent rate-limit state, so that changing one tier's policy or one client's usage does not unintentionally affect other clients or tiers.

**Why this priority**: Independent state and configurable policy make the service useful for more than a single client and allow operators to tune tiers without changing the limiting rules.

**Independent Test**: Exhaust one client's bucket, verify another client can still use its own bucket, then change one tier's configured quota and verify the other tiers retain their configured values.

**Acceptance Scenarios**:

1. **Given** two clients on the same tier with full two-token buckets, **When** the first client consumes both tokens, **Then** a request from the second client is allowed with one token remaining.
2. **Given** default quotas of 10, 100, and 1000 requests per minute, **When** the free-tier quota is configured as 2 requests per minute, **Then** the effective quotas are 2, 100, and 1000 requests per minute for free, pro, and enterprise respectively.

### User Story 3 - Receive clear errors for invalid requests (Priority: P3)

As a calling system, I want invalid requests to receive a clear client error without changing rate-limit state, so that I can correct the request without accidentally using the client's quota.

**Why this priority**: Clear validation protects valid clients from malformed calls and makes integration failures diagnosable.

**Independent Test**: Submit missing, malformed, and unknown-tier requests, verify each receives its specified error, then send a valid request for that client and verify its bucket is unchanged.

**Acceptance Scenarios**:

1. **Given** a request without a client identifier, **When** it is submitted, **Then** the service returns HTTP 400 with `{"error":"client_id is required"}` and does not create or consume a bucket.
2. **Given** a request with an unknown tier, **When** it is submitted, **Then** the service returns HTTP 400 with `{"error":"tier must be one of: free, pro, enterprise"}` and does not create or consume a bucket.
3. **Given** a malformed JSON body, **When** it is submitted, **Then** the service returns HTTP 400 with `{"error":"request body must be valid JSON"}` and does not create or consume a bucket.

### Edge Cases

- A client identifier that is absent, null, empty, or not a string is invalid and must not consume tokens.
- A tier value is case-sensitive and must match exactly one of `free`, `pro`, or `enterprise`.
- A missing, null, non-string, or unsupported `tier` returns HTTP 400 with `{"error":"tier must be one of: free, pro, enterprise"}` and does not create or consume a bucket.
- A request with extra JSON fields is accepted; only the defined client identifier and tier fields affect the decision.
- Refill never raises a bucket above its configured capacity, including after a long idle period.
- A retry delay is rounded up to the next whole second so the caller is not told to retry before a full token is available.
- Simultaneous checks for the same client must not allow more requests than the available tokens.
- A service restart clears all client buckets; no rate-limit state survives the process lifetime.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The service MUST expose `POST /rate-limit/check`, accepting a JSON object with `client_id` and `tier` fields.
- **FR-002**: The service MUST provide the default tier quotas of 10 requests per minute for `free`, 100 for `pro`, and 1000 for `enterprise`; these values MUST be configurable separately from the token-bucket decision rules.
- **FR-003**: The service MUST maintain independent rate-limit state for each client identifier; a request for one client MUST NOT consume another client's tokens.
- **FR-004**: The service MUST allow a valid request when its client's bucket has at least one token, consume exactly one token for each allowed request, and report the remaining whole-token count.
- **FR-005**: The service MUST refill each bucket over elapsed time at its configured per-minute quota, never exceed its configured capacity, reject requests when fewer than one token is available, and report the whole-second retry delay rounded up to when one token will be available.
- **FR-006**: The service MUST reject a missing, null, non-string, or unsupported tier, absent or invalid client identifier, malformed JSON, or a request body that is not a JSON object with HTTP 400 and a clear JSON error; validation failures MUST NOT create or consume a client's bucket. A missing, null, non-string, or unsupported `tier` MUST produce `{"error":"tier must be one of: free, pro, enterprise"}`. A missing or null `client_id` MUST produce `{"error":"client_id is required"}`; an empty or non-string `client_id` MUST produce `{"error":"client_id must be a non-empty string"}`; a non-object JSON body MUST produce `{"error":"request body must be a JSON object"}`; malformed JSON MUST produce `{"error":"request body must be valid JSON"}`.
- **FR-007**: An allowed decision MUST return HTTP 200 and a JSON response containing `allowed: true`, `limit`, and `remaining`. A rate-limited decision MUST return HTTP 429 and a JSON response containing `allowed: false`, `limit`, `remaining: 0`, and `retry_after_seconds`; the response MUST include `Retry-After`, `X-RateLimit-Limit`, and `X-RateLimit-Remaining` headers with the corresponding values.
- **FR-008**: Tier configuration MUST define each tier's quota and bucket capacity independently of the algorithm. The default quotas are specified in FR-002; the default capacities are 10 tokens for `free`, 100 for `pro`, and 1000 for `enterprise`. Capacities MUST remain configurable separately from the token-bucket decision rules.
- **FR-009**: Rate-limit state MUST be held in memory only and MUST be cleared when the service process stops or restarts.
- **FR-010**: The service MUST make each check atomic with respect to other checks for the same client, so concurrent requests cannot consume the same token more than once.
- **FR-011**: The initial service MUST exclude authentication, persistent storage, distributed or multi-instance coordination, dashboards, and user interfaces.

### Non-Functional Requirements

- **NFR-001**: The service MUST make the same decision for the same configured inputs and elapsed-time sequence; token refill and retry calculation MUST not depend on unobservable timing.
- **NFR-002**: The request and response contract MUST clearly distinguish allowed requests, rate-limited requests, and invalid requests using the statuses, fields, and headers defined in FR-006 and FR-007.
- **NFR-003**: The implementation MUST remain a single-instance, in-memory service for this release; it MUST NOT imply that decisions are shared across service instances.
- **NFR-004**: Tier quota changes MUST NOT require changes to the token-bucket decision rules, and changing one tier's quota MUST NOT alter another tier's quota.

### Acceptance Criteria

- **AC-001 (FR-002, FR-008, NFR-004)**: With no overrides, the configured quotas are exactly `free=10`, `pro=100`, and `enterprise=1000` requests per 60 seconds, and the capacities are exactly `free=10`, `pro=100`, and `enterprise=1000` tokens. With only `free` quota overridden to 2 requests per 60 seconds, the effective quotas are exactly `free=2`, `pro=100`, and `enterprise=1000`, and capacities remain `free=10`, `pro=100`, and `enterprise=1000` unless separately overridden.
- **AC-002 (FR-001, FR-003, FR-004)**: With a test configuration of `free` quota 2 per 60 seconds and capacity 2, sending `POST /rate-limit/check` with JSON `{"client_id":"alpha","tier":"free"}` twice returns HTTP 200 with `allowed=true`, `limit=2`, and `remaining` values 1 then 0; the third identical request returns HTTP 429 with `allowed=false`, `limit=2`, `remaining=0`.
- **AC-003 (FR-003)**: With the same configuration, after client `alpha` consumes both tokens, the first check for client `beta` returns HTTP 200 with `allowed=true`, `limit=2`, and `remaining=1`.
- **AC-004 (FR-005, FR-008)**: With quota 2 per 60 seconds and capacity 2, after consuming both tokens, a check at 29 elapsed seconds returns HTTP 429 with `retry_after_seconds=1` and `Retry-After: 1`; at 30 elapsed seconds, the next check returns HTTP 200 with `remaining=1`.
- **AC-005 (FR-005, FR-007)**: With quota 2 per 60 seconds and capacity 2, an immediate check after consuming both tokens returns HTTP 429, body fields `allowed=false`, `limit=2`, `remaining=0`, and `retry_after_seconds=30`, and headers `Retry-After: 30`, `X-RateLimit-Limit: 2`, and `X-RateLimit-Remaining: 0`.
- **AC-006 (FR-006)**: For requests with missing `client_id`, null `client_id`, empty `client_id`, non-string `client_id`, missing `tier`, null `tier`, non-string `tier`, unknown tier, a top-level JSON array, and malformed JSON, respectively, the service returns HTTP 400 with the exact error bodies `{"error":"client_id is required"}`, `{"error":"client_id is required"}`, `{"error":"client_id must be a non-empty string"}`, `{"error":"client_id must be a non-empty string"}`, `{"error":"tier must be one of: free, pro, enterprise"}`, `{"error":"tier must be one of: free, pro, enterprise"}`, `{"error":"tier must be one of: free, pro, enterprise"}`, `{"error":"tier must be one of: free, pro, enterprise"}`, `{"error":"request body must be a JSON object"}`, and `{"error":"request body must be valid JSON"}`.
- **AC-007 (FR-006)**: With `free` configured as quota 2 per 60 seconds and capacity 2, after any sequence of invalid requests for client `alpha`, its first subsequent valid check returns HTTP 200 with `remaining=1`; invalid requests do not create or consume a token.
- **AC-008 (FR-007)**: Every allowed decision returns HTTP 200 and includes exactly the decision fields `allowed=true`, the tier's configured `limit`, and the post-consumption `remaining` value. Every rate-limited decision returns HTTP 429 and includes the fields and three headers specified in AC-005, with values matching that decision.
- **AC-009 (FR-005)**: With quota 2 per 60 seconds and capacity 2, after 60 seconds with no requests, the bucket is still exactly 2 tokens; the next two requests are allowed and the third is rejected with `retry_after_seconds=30`.
- **AC-010 (FR-009)**: With `free` configured as quota 2 per 60 seconds and capacity 2, after one allowed request and a service restart, the first request for that client is allowed with `remaining=1`, demonstrating that prior process state was not retained.
- **AC-011 (FR-010)**: With `free` configured as quota 2 per 60 seconds and capacity 2, 20 simultaneous checks for a previously unused client produce exactly 2 HTTP 200 decisions and 18 HTTP 429 decisions.
- **AC-012 (FR-011)**: The service accepts a request to `POST /rate-limit/check` without credentials, exposes no other service operation, and provides no dashboard or user interface. The deployment has one instance, and a restart clears all rate-limit state as verified by AC-010.

### Key Entities

- **Client**: The caller-identified subject whose requests share one independent bucket.
- **Tier**: A named service level (`free`, `pro`, or `enterprise`) with a configurable request quota and bucket capacity.
- **Token bucket**: A client's current whole and fractional token balance, refill rate, capacity, and time of last refill; one whole token authorizes one request.
- **Rate-limit decision**: The result for one check, including allow/reject status, configured limit, remaining capacity, and (when rejected) retry delay.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: The three default tier quotas are exactly 10, 100, and 1000 requests per 60 seconds, and all three are independently configurable.
- **SC-002**: In configured scenarios, the number of allowed requests never exceeds available whole tokens; each allowed request consumes exactly one token and each rejected request consumes zero tokens.
- **SC-003**: For all valid allow and reject scenarios, 100% of returned statuses, response values, and required headers match the values defined in the acceptance criteria.
- **SC-004**: For every invalid-request scenario in AC-006, the service returns the exact specified error response and leaves the client's available tokens unchanged.
- **SC-005**: Rate-limit state is cleared on restart, and all decisions are made using only one service instance's in-memory state.

## Assumptions

- A new client's bucket starts full at its configured capacity.
- The configured per-minute quota refills evenly over time (for example, two tokens per minute yields one token every 30 seconds, including fractional progress between whole tokens).
- Retry delays are reported as whole seconds, rounded up to the next second.
- Tier configuration is supplied when the service starts; no runtime configuration management interface is included.
- Client identifiers are non-empty strings and are treated as case-sensitive.
- Extra fields in an otherwise valid request are ignored.
- The only supported tier names are `free`, `pro`, and `enterprise`.

## Unresolved Questions

- None. Q1 was resolved: the default capacities equal their per-minute quotas (free=10, pro=100, enterprise=1000 tokens); capacities may still be configured separately.
