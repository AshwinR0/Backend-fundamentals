# Authentication and Authorization for Backend Engineers

## 1. The Fundamental Definitions
At the core of application security are two distinct but interconnected concepts:
*   **Authentication (AuthN):** The mechanism of assigning an identity to a subject. It answers the question, *"Who are you in a given context?"* (e.g., logging into a platform).
*   **Authorization (AuthZ):** The process of determining a subject's capabilities. It answers the question, *"What can you do in that context?"* (e.g., assessing your permissions once logged in).

---

## 2. The Historical Evolution of Authentication
Understanding modern security requires understanding the historical problems each era solved:

*   **Pre-Industrial (Implicit Trust):** Identity relied on human contextual trust and recognition (e.g., a village elder vouching for someone). As society grew, this failed to scale.
*   **Medieval Era (Explicit Possession):** Society shifted to physical tokens, like wax seals on documents, marking early cryptographic thinking. However, these were prone to forgery, marking the first instances of "authentication bypass attacks".
*   **Industrial Era (Shared Secrets):** The telegraph necessitated secure message validation using pre-agreed passphrases (static passwords), shifting the paradigm from "something you possess" to "something you know".
*   **Mainframe Era (1960s):** MIT's Project MAC introduced passwords for multi-user systems. Initially stored in plaintext, a critical vulnerability was exposed when a user accidentally printed the password file. This incident birthed the concept of secure password storage via **hashing** (irreversible, fixed-length representations).
*   **1970s Cryptography:** Diffie-Hellman introduced asymmetric cryptography (Public Key Infrastructure), and Kerberos introduced ticket-based authentication via trusted third parties.
*   **1990s Internet Boom:** Simple passwords became vulnerable to brute-force and dictionary attacks. This led to **MFA (Multi-Factor Authentication)**, combining layers: Something you know (passwords), something you have (OTP cards), and something you are (biometrics).
*   **21st Century & Beyond:** Cloud computing and APIs demanded robust frameworks like OAuth 2.0 and JWTs. The future points toward decentralized identity (blockchain), behavioral biometrics, and "Post-Quantum Cryptography" to secure data against massively powerful quantum computers.

---

## 3. The Three Core Components of Modern Auth
Modern authentication heavily relies on three foundational tools:

### Sessions
By design, HTTP is stateless (it forgets you after every request). As the web became dynamic (e.g., e-commerce carts), developers created "sessions" to give the server memory. A Session ID is generated, stored alongside user data in a persistent server store (like a Database or Redis), and sent to the client. Sessions evolved from file-based storage to databases, and now to in-memory distributed stores like Redis or Memcached for high scalability.

### JWTs (JSON Web Tokens)
As applications scaled globally, looking up session data in a database for millions of users introduced latency and synchronization bottlenecks. Introduced around 2015, JWTs solve this by being entirely self-contained and stateless. 
*   **Structure:** Contains a Header (metadata/algorithm), Payload (claims like user ID `sub` and issued-at time `iat`), and a Cryptographic Signature.
*   **Pros & Cons:** JWTs eliminate server-side storage costs and are highly portable across microservices. However, revoking a JWT is difficult because it is stateless; if a token is compromised, the attacker can impersonate the user until the token naturally expires.

### Cookies
A browser feature allowing a server to store a string in the client's browser. Cookies automatically attach themselves to every subsequent request sent back to the server that created them. This automates the process of identifying a user on every request.

**Text Illustration: The JWT Structure**
```text
[ Header ]      (Algorithm: HS256, Type: JWT)
   +
[ Payload ]     (Sub: "user_123", Role: "admin", IAT: 1615300000)
   +
[ Signature ]   (Cryptographic hash of Header + Payload + Server Secret Key)
-------------------------------------------------------------------------
= Base64 Encoded Token: eyJhbGci...eyJzdWIi...SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV
```

---

## 4. The 4 Major Types of Authentication
*   **Stateful Authentication:** The traditional approach. The client sends credentials; the server verifies them, saves a session in Redis, and returns a Session ID via a cookie. 
    *   *Use Case:* Standard Web Apps. Pros include centralized control and easy, immediate revocation of compromised accounts. Cons are scaling complexity.
*   **Stateless Authentication:** The client sends credentials; the server verifies and returns a signed JWT. The client stores it and sends it back in the `Authorization` header for subsequent requests.
    *   *Use Case:* APIs and distributed systems/mobile apps. Pros include massive scalability without database hits. A *Hybrid Approach* is sometimes used, utilizing JWTs for speed but maintaining a database blacklist to temporarily block malicious tokens.
*   **API Key Authentication:** Designed specifically for programmatic, non-human interaction (Machine-to-Machine).
    *   *Use Case:* A backend server accessing another backend service (e.g., your server automatically calling ChatGPT's API to summarize text). It eliminates the need for manual login flows.
*   **OAuth 2.0 & OpenID Connect (OIDC):** Solves the "Delegation Problem". Historically, if an app wanted access to your Google contacts, you had to hand over your actual password, giving them terrifying, un-revocable full access. 
    *   *OAuth 2.0:* Exists purely for **Authorization**. It allows a user to grant specific permissions (via scoped tokens) to a third-party app without sharing their password. It introduced various flows depending on the device (Authorization Code flow for servers, Device Code flow for Smart TVs).
    *   *OpenID Connect (OIDC):* Built on top of OAuth 2.0 to solve **Authentication**. It introduces an "ID Token" (a JWT containing the user's identity). This technology powers the ubiquitous "Sign in with Google" or "Sign in with Facebook" buttons.

**Text Illustration: The OIDC "Sign In With Google" Flow**
```text
[ Notetaking App ] ──(1. Redirects to Google)──> [ Google Auth Server ]
        ▲                                                │
        │                                        (2. User logs in & grants access)
        │                                                │
        └─(3. Returns Auth Code + ID Token)──────────────┘
                    (JWT with Identity)
```

---

## 5. Authorization: Role-Based Access Control (RBAC)
Once a user is authenticated, authorization ensures they can only access what they are allowed to. A common pattern is RBAC.
*   **How it works:** Users are assigned specific roles (e.g., User, Moderator, Admin). Each role is mapped to specific capabilities (e.g., Admins can read, write, and access a deleted items "Dead Zone"; Users can only read).
*   **Implementation:** When a request hits the server, middleware intercepts it, deduces the user's role from their token/session, and blocks unauthorized actions by returning a `403 Forbidden` HTTP status code. Hardcoding a secret "god mode string" in an API to bypass permissions is a massive security flaw and an anti-pattern.

---

## 6. Critical Security Implementation Practices
*   **Generic Error Messages:** Never send client-friendly errors like "User not found" or "Incorrect password" during login. This tells an attacker exactly which part of the credential pair is correct, assisting brute-force attacks. Always return a generic `"Authentication failed"`.
*   **Defeating Timing Attacks:** When validating a login, an invalid username fails instantly, whereas a valid username with a bad password fails slowly (because the server takes time to hash the password). Attackers measure this millisecond delay to harvest valid usernames. **Solution:** Use constant-time comparison operations, or simulate a fake network delay (e.g., `setTimeout` or `time.sleep()`) so that all login failures take the exact same amount of time.

---

## 7. Supplemental "Good-to-Have" Points (External Information)
*   **Salting Passwords:** The video mentions hashing passwords. In practice, backend engineers never *just* hash a password. They add a "Salt" (a unique, random string) to the password before hashing it. This ensures that even if two users have the password "password123", their resulting hashes in the database look completely different, protecting against pre-computed dictionary attacks (Rainbow Tables).
*   **HttpOnly and Secure Cookie Flags:** When the video notes that a session ID is stored in a cookie so JavaScript cannot access it, it is referring specifically to setting the `HttpOnly` flag on the cookie. Furthermore, the `Secure` flag is almost always used in production to ensure the cookie is only transmitted over encrypted HTTPS connections.
*   **WebAuthn (Passwordless Auth):** Mentioned briefly in the video as the future. Web Authentication utilizes the secure enclave on your physical device (like Apple's FaceID, TouchID, or a YubiKey hardware token). It entirely removes passwords from the equation, generating a public/private key pair. This is practically immune to phishing since there is no password for the user to accidentally give away.
*   **The Zero Trust Architecture:** Another modern concept mentioned briefly. Historically, systems assumed that any traffic *inside* the corporate network was safe (the "castle-and-moat" model). Zero Trust assumes the network is already compromised. It requires strict, continuous authentication and authorization for every single user, device, and API call, regardless of whether they are sitting inside the corporate office or logging in remotely.

![Alt text](./images/06%20Authentication%20and%20Authorization.png)