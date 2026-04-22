# Deep Dive: Authentication and Authorization in Backend Engineering
## 1. Core Definitions: The Two Pillars of Security

Understanding the difference between these two concepts is the foundation of backend security:

*   **Authentication (AuthN):** This is the mechanism used to assign a specific identity to a subject. It answers the fundamental question: **"Who are you in this given context?"**
*   **Authorization (AuthR):** This is the process of determining a user’s specific capabilities and access levels. It answers the question: **"What are you allowed to do in this particular context?"**

---

## 2. The Historical Evolution of Authentication

The methods we use to prove identity have evolved alongside human technology:

*   **Pre-Industrial and Medieval Eras:** Authentication was originally based on personal recognition and human trust. As societies grew, **wax seals** became the first "authentication tokens." However, these were susceptible to forgery—the earliest form of a bypass attack.
*   **Industrial Era:** The invention of the telegraph required distant operators to verify each other using static **passphrases** or shared secrets. This shifted authentication to "something you know."
*   **Mainframes and Early Computing (1960s):** MIT’s Project MAC introduced passwords for multi-user systems. Initially, these were stored in plain text. When a plain-text password file was accidentally printed, exposing every user, it led to the invention of **hashing**—using mathematical functions to transform passwords into secure, irreversible, fixed-length representations.
*   **1970s Cryptography:** The introduction of the Diffie-Hellman key exchange and asymmetric cryptography laid the groundwork for modern token-based systems like Kerberos.
*   **Modern Era:** The 1990s saw the rise of **Multi-Factor Authentication (MFA)**, which combines something you know (password), something you have (an OTP device or phone), and something you are (biometrics like fingerprints). The 21st century brought API-based architectures, leading to OAuth, JWTs, and decentralized identities.

---

## 3. Key Authentication Components

### A. Sessions
Since HTTP is a **stateless protocol** (it has no memory of previous interactions), developers use sessions to create stateful experiences.
*   **Mechanism:** The server creates a unique **Session ID**, stores it in a persistent store (like a database or Redis), and sends the ID to the client as a cookie.
*   **Usage:** The client includes this cookie in every subsequent request, allowing the server to look up the user's data in its memory.

### B. JSON Web Tokens (JWTs)
JWTs emerged to solve the problem of scaling sessions for millions of users across distributed servers.
*   **Structure:** They are **stateless, self-contained tokens** consisting of three parts:
    1.  **Header:** Contains metadata and the signing algorithm used.
    2.  **Payload:** Contains "claims" such as the User ID (`sub`), roles, and the time the token was issued.
    3.  **Signature:** A cryptographic seal that ensures the token hasn't been tampered with.
*   **Pros & Cons:** They are highly portable and reduce server storage costs. However, they are difficult to revoke; to cancel a JWT before it expires, you typically have to change the server's global secret key (logging out everyone) or maintain a blacklist.

### C. Cookies
Cookies are a browser-side mechanism. When a server "sets" a cookie, the browser automatically attaches it to every future request made to that specific server. This makes them the primary vehicle for transporting Session IDs and JWTs securely.

---

## 4. Major Types of Modern Authentication

1.  **Stateful Authentication:** Relies on Sessions and persistent storage (like Redis).
    *   *Benefits:* Offers centralized control and real-time tracking of active users. Token revocation is easy (just delete the session from the DB).
    *   *Drawbacks:* Harder to scale in massive, distributed microservice architectures.
2.  **Stateless Authentication:** Relies on JWTs verified via a secret key.
    *   *Benefits:* Extremely scalable; the server doesn't need to check a database for every request.
    *   *Drawbacks:* Revoking a single compromised token is difficult without a hybrid approach.
3.  **API Key Based Authentication:** Uses a secure, random string (API Key) to allow programmatic access. This is designed for **machine-to-machine communication**, where one server needs to interact with another without a human login form.

---

## 5. Detailed Breakdown: OAuth 1.0 vs. OAuth 2.0

### The Delegation Problem
Before OAuth, if a "Trip Planner" app wanted to see your "Google Calendar," you had to give the Trip Planner your actual Google password.
*   **The Risk:** The Trip Planner could now delete your emails, change your password, and lock you out. There was no way to limit their access to *just* the calendar.

### OAuth 1.0: The Cryptographic Predecessor
Created in 2007, OAuth 1.0 (often called "3-Legged OAuth") focused on security through **digital signatures.**

*   **How it works (The Handshake):**
    1.  **Request Token:** The app asks the server for a temporary token.
    2.  **User Authorization:** The app sends the user to the server to login and say "Yes, I allow this."
    3.  **Exchange:** The app exchanges the temporary token for a long-term **Access Token.**
*   **The "Secret" Key:** Every request required the client to use a "Token Secret" to generate a unique signature. The secret itself was never sent over the internet.
*   **Example:** Early versions of the Twitter API. If an app wanted to post a tweet for you, it had to perform a complex mathematical "dance" to sign every single request.
*   **The Downside:** It was extremely difficult to code. If your "Signature Base String" had one extra space or the wrong capitalization, the whole request failed, making it a nightmare for mobile developers.

### OAuth 2.0: The Industry Standard
Released in 2010, OAuth 2.0 moved away from signatures and toward **Bearer Tokens.**

*   **How it works:** It uses **HTTPS** as the security layer. Because the "pipe" is encrypted, we can simply send a "Bearer Token" (a string of characters). If you "bear" (hold) the token, you have the power.
*   **Example:** Spotify using your Facebook friend list. Spotify asks Facebook for a token. Facebook asks you for permission. Once granted, Spotify just sends the token `Bearer abc123xyz` in the header to get your friends.

### Why was OAuth 2.0 introduced? (Key Differences)
1.  **Support for Non-Browser Apps:** OAuth 1.0 assumed a web browser was always involved. OAuth 2.0 added "Grant Types" for mobile apps and Smart TVs (Device Flow).
2.  **Developer Experience:** By eliminating complex signatures, developers could implement it in minutes rather than days.
3.  **Performance:** OAuth 1.0 required the server to do heavy math to verify signatures for *every* request. OAuth 2.0 tokens (especially if they are JWTs) are much faster to verify.
4.  **Short-lived Tokens:** OAuth 2.0 introduced **Refresh Tokens**, allowing the "Access Token" to expire every hour for better security, while the app can get a new one without bothering the user.

---

## 6. OpenID Connect (OIDC): The Identity Layer

OAuth 2.0 is a **Key**, but OIDC is an **ID Card.**

*   **OAuth 2.0 (Authorization):** It gives the app an **Access Token** to do things (like "Post to Facebook"). It doesn't tell the app *who* you are.
*   **OIDC (Authentication):** It sits on top of OAuth 2.0 and provides an **ID Token** (a JWT). This token contains your name, email, and profile picture.
*   **Example:** "Sign in with Google" on a news site.
    *   The news site doesn't care about your Google Drive or Gmail (OAuth).
    *   The news site just wants to know your email address so they can create an account for you (OIDC).
*   **Standardization:** Before OIDC, every company (Facebook, Twitter, Google) had their own way of sending user info. OIDC created one standard format (`iss`, `sub`, `email`, `aud`) that everyone uses.

---

## 7. Authorization: Role-Based Access Control (RBAC)

RBAC is the most common way to manage "What can you do?" in a backend. It organizes permissions into logical buckets.

*   **The Components:**
    1.  **Permissions:** The smallest unit (e.g., `edit_post`, `delete_comment`, `invite_user`).
    2.  **Roles:** A group of permissions (e.g., an "Editor" role has `edit_post` but not `invite_user`).
    3.  **User-Role Mapping:** You tell the database "User #5 is an Editor."
*   **Example (Corporate Dashboard):**
    *   **User A (Role: Employee):** Can view their own payslip.
    *   **User B (Role: Manager):** Can view payslips for their whole team.
    *   **User C (Role: Admin):** Can change anyone's salary.
*   **Why use it?** If you have 1,000 users, you don't want to assign 50 permissions to each one manually. You just assign them a "Role." If you decide Editors can now `delete_post`, you change it once in the Role, and all 1,000 Editors get the update instantly.

---

## 8. Defending Against Timing Attacks

A timing attack is a "side-channel" attack where a hacker uses a stopwatch to steal information.

### The Vulnerability: String Comparison
In most code, `if (input == "SecretPassword")` is fast. If the first letter is wrong, the computer stops immediately (**Early Exit**).
*   **Scenario:** A hacker tries to guess a username.
    *   Username "Admin" exists $\rightarrow$ Server hashes the password (takes **200ms**).
    *   Username "FakeUser" doesn't exist $\rightarrow$ Server returns "User not found" (takes **5ms**).
*   **The Hack:** The hacker now knows "Admin" is a valid account because the server took longer to answer.

### The Defenses:
1.  **Constant-Time Comparisons:** Use a library function (like `crypto.timingSafeEqual`) that compares the entire string even if it finds a mistake early. This ensures the comparison always takes, for example, exactly **2ms**.
2.  **Artificial Jitter/Delays:** If a user is not found, force the server to wait.
    *   **Example:**
        ```javascript
        if (!user) {
            await simulatePasswordHash(); // Pretend to do the heavy work
            return error("Invalid credentials");
        }
        ```
    *   By "faking" the work, the response for a non-existent user now takes **200ms**, exactly matching a real user. The hacker can no longer tell the difference.

---

## 9. Practical Workflow Case Studies
*(Refer to previous notes for Cases 1-4: Stateful, Stateless, Hybrid, and API Keys).*

### Case 5: OAuth 1.0 (Detailed Flow)
1.  **Request:** App asks for a "Request Token."
2.  **Redirect:** User goes to the provider (e.g., Twitter) and signs in.
3.  **Verification:** User is sent back to the app with a "Verifier."
4.  **Final Exchange:** App sends the Request Token + Verifier + **Secret Key Signature** to get the final Access Token.
*   **Key Point:** Relies on the client's ability to do complex math (HMAC signatures).

### Case 6: OpenID Connect (OIDC) Workflow
1.  **Sign-In:** User clicks "Sign in with Google."
2.  **Consent:** User approves sharing "Profile" and "Email."
3.  **Token Delivery:** Google sends back an **ID Token** (JWT).
4.  **Parsing:** The backend decodes the JWT and reads: `email: "user@gmail.com", name: "John Doe"`.
5.  **Action:** The backend automatically logs the user in without ever asking them to create a password for *this* site.

### Case 7: Role-Based Access Control (RBAC) Workflow
1.  **Request:** A user calls `DELETE /api/posts/101`.
2.  **Identity:** Middleware reads the JWT and finds `role: "Guest"`.
3.  **Permission Check:** The system looks at the **Permission Matrix**: "Can a Guest delete posts? -> NO."
4.  **Response:** The request is blocked with a `403 Forbidden` status code before the database is even touched.

### Case 8: Defending Against Timing Attacks
1.  **The Goal:** Make "User Not Found" and "Wrong Password" look identical to a stopwatch.
2.  **Implementation:** If the username lookup fails, run a "Dummy Hash."
3.  **Result:** Both failure paths now consume roughly the same CPU cycles and time, blinding the attacker to the existence of valid usernames.