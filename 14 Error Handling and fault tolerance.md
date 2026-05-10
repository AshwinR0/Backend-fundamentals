# Error Handling and Building Fault-Tolerant Systems

## 1. The Fault-Tolerant Mindset
In backend development, errors are an inevitable reality—database queries will fail, external APIs will time out, users will submit bad data, and edge cases will occur. Building fault-tolerant systems is not just about using specific tools or frameworks; it requires a proactive mindset where developers anticipate failures and prepare mechanisms to detect, contain, and recover from them gracefully.

---

## 2. The 5 Major Types of Backend Errors

### Logic Errors
The most dangerous and difficult-to-detect errors because they do not crash the application. The code runs completely fine but produces incorrect business results (e.g., an e-commerce algorithm applying a discount twice, causing the platform to bleed money silently for months). 
*   **Causes:** Misunderstanding product requirements, implementing complex algorithms incorrectly, or failing to anticipate specific user behaviors.

### Database Errors
Can bring an entire system to a halt. 
*   **Connection Errors:** Occur when the network is down, the DB is overloaded, or the TCP connection pool is completely exhausted, resulting in 500 Server Errors.
*   **Constraint Violations:** Attempting actions that break database rules, such as inserting a duplicate email (`Unique Constraint`) or referencing a customer ID that doesn't exist in an orders table (`Foreign Key Constraint`).
*   **Query Errors & Deadlocks:** Caused by malformed SQL typos or circular dependency locks where operations wait on each other endlessly.

### External Service Errors
Modern SaaS apps rely heavily on third parties (payment processors, Resend for emails, Auth0 for auth, AWS S3). These are points of failure outside of your direct control.
*   **Network Issues:** Connection timeouts, DNS failures, or routing partitions.
*   **Rate Limiting (429 Too Many Requests):** Triggered when you hit an external API too rapidly, causing them to block your requests to prevent abuse.
*   **Service Outages:** Cloud provider downtimes that require you to have fallbacks (like a secondary cache) to ensure graceful degradation.

### Input Validation Errors
The first line of defense, triggered when users send bad or malicious data. Includes format validation (bad emails), range validation (string too long), and required field validation. These are the easiest to anticipate and usually return a `400 Bad Request`.

### Configuration Errors
Occurs when moving between environments (Dev -> Staging -> Prod) and forgetting to inject required environment variables (like an OpenAI API key). 
*   **Best Practice:** Validate all required configuration variables at the absolute start of the application. It is much better to crash the app immediately upon deployment (**fail-fast**) than to let it run and randomly throw 500 errors to users at runtime.

---

## 3. Proactive Prevention: Health Checks & Observability
The best error handling starts before the error actually happens.

*   **Health Checks:** Expose a `/health` endpoint to return a 200 OK status. Do not just check if the HTTP server is running; run "representative queries" to test database query performance, and send test tokens to verify external auth providers are actually responsive.
*   **Monitoring Metrics:** Do not only track raw error logs. Monitor performance metrics (like sudden latency spikes or a steep drop in successful transactions), as performance degradation is often the earliest tell-tale sign of an impending system crash.
*   **Structured Logging:** Log in structured formats (like JSON) and aggregate them using tools like Grafana or Loki so they can be easily parsed and searched.

---

## 4. Error Response and Recovery Strategies
When an error happens, your immediate response determines if it stays a minor issue or becomes a major incident.

*   **Recoverable Errors:** For issues like network timeouts or rate limits (e.g., failing to send an email), implement an **Exponential Backoff** strategy (retrying after 1 min, then 2 mins, then 4 mins). Ensure your retry logic doesn't overwhelm an already stressed system.
*   **Non-Recoverable Errors:** Implement containment and graceful degradation. If a service is totally down, disable that specific non-essential feature or serve cached fallback data instead of crashing the whole UI.
*   **Error Boundaries (Decoupling):** Use separate processes and message queues (like RabbitMQ) to decouple services. This asynchronous communication ensures a bug in one service doesn't cascade and kill another.

---

## 5. The Global Error Handler (The Final Safety Net)
A global error handling middleware is a centralized location that catches any error bubbling up from the application logic, avoiding redundant error-handling code scattered across the codebase.

### Text Illustration: Error Bubbling and Global Handling
```text
[ Client Request ]
       │
[ Global Error Handler Middleware ] <── (Catches EVERYTHING bubbling up)
       │                                  ├── IF Validation Error -> Returns 400
       ▼                                  ├── IF Unique DB Error  -> Returns 400
[ Controller / Handler ]                  ├── IF No Rows DB Error -> Returns 404
       │ (Throws Validation Errors)       └── IF Unknown Error    -> Returns 500
       ▼
[ Service Layer ]
       │ (Throws Business Logic Errors)
       ▼
[ Repository Layer ]
       (Throws Database Errors: Unique Constraint, Foreign Key, No Rows)
```

*   **Standardizing Responses:** If the Repository layer throws a "No Rows" error when querying an ID, it bubbles up to the global handler, which intercepts it and transforms it into a clean `404 Not Found` response. If a "Unique Constraint" is thrown, it is intercepted and returned as a `400 Bad Request` ("Item already exists").
*   **Avoiding Generic Failures:** Without a global handler, unhandled specific errors usually default to a generic `500 Internal Server Error`, creating a terrible user experience.

---

## 6. Security in Error Handling
Error messages can be weaponized by attackers if not properly secured.

*   **Preventing Information Leakage:** Never leak internal database details (table names, column constraints, SQL syntax) in HTTP responses. Malicious users can use these clues to craft advanced SQL injection attacks. If an error reaches the 500-level, strictly return a generic "Something went wrong" message.
*   **Preventing Username Enumeration:** In authentication endpoints (login/signup), never return specific messages like "User with this email does not exist" or "Incorrect password". Attackers will use automation to verify which emails exist in your system and then brute-force those specific accounts. Always return a generic `"Invalid email or password"`.
*   **Secure Logging:** Never log sensitive Personally Identifiable Information (PII) such as plaintext passwords, credit card numbers, or API keys. If log management platforms (ELK, Datadog) suffer a data breach, your users' data goes with it. Log internal `user_id`s or correlation IDs instead.

---

## 7. Supplemental "Good-to-Have" Points (External Context)
*Note: The following concepts expand upon the video but rely on standard backend industry knowledge.*

*   **The Circuit Breaker Pattern:** Highly relevant to external service errors. If your backend detects that an external service (like a payment gateway) has failed 5 times in a row, the "circuit breaker" trips. Instead of continuing to hit the dead service and racking up timeouts, your backend instantly fails fast for all subsequent requests for a set cooldown period.
*   **Correlation IDs for Tracing:** When logging errors in a microservices architecture, a single user click might trigger 5 different backend services. Attaching a unique `X-Correlation-ID` header to the initial request and passing it along ensures that when an error occurs deep in the system, you can filter your logs by that ID and trace the exact path the request took.
*   **OWASP Top 10:** The industry standard for web security. "Security Misconfiguration" (like leaking detailed stack traces/errors to production users) consistently ranks as one of the most critical vulnerabilities.