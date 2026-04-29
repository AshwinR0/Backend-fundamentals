# Complete REST API Design

## 1. The History and Need for API Standards
When developers build APIs, they often struggle with questions like whether routes should be plural or singular, when to use PUT vs. PATCH, or how to design custom actions. These confusions stem from the fact that modern web development heavily uses Single-Page Applications (SPAs), whereas the foundational web standards were originally built for Multi-Page Applications (MPAs). By adhering to consistent industry standards, backend engineers can eliminate guesswork and focus entirely on business logic.

*   **The Origins:** In 1990, Tim Berners-Lee created the World Wide Web, inventing URIs, HTTP, HTML, and the first web server and browser.
*   **The Scalability Crisis:** As the internet grew exponentially, it faced a breakdown. The original design was not equipped for massive scale. 
*   **The Solution:** In 1993, Roy Fielding (co-founder of Apache) proposed a set of architectural constraints to solve this scalability issue. He later formalized this in his 2000 PhD dissertation, naming the architecture REST (Representational State Transfer).

---

## 2. Roy Fielding’s 6 Constraints of REST
To make the web scalable, an architecture must strictly follow these constraints:

1.  **Client-Server:** A strict separation of concerns. The client handles UI/UX, while the server handles data storage and business logic.
2.  **Uniform Interface:** A standardized way for components to communicate. It includes resource identification, manipulation through representation, self-descriptive messages, and Hypermedia (HATEOAS).
3.  **Layered System:** A hierarchical architecture where each layer only interacts with the immediate layer below it. This allows the seamless injection of load balancers and proxy servers.
4.  **Cache:** Server responses must explicitly label themselves as cacheable or non-cacheable to reduce server load and improve speed.
5.  **Statelessness:** The server does not store any client context between requests. Every single request must contain all the information necessary for the server to process it.
6.  **Code on Demand (Optional):** Servers can temporarily extend client functionality by sending executable code (like JavaScript).

---

## 3. Decoding "REST" (Representational State Transfer)
*   **Representational:** A single resource (like a user or a book) can be represented in multiple formats depending on the client's needs—such as JSON for another server, or HTML for a web browser.
*   **State:** The current condition, attributes, or properties of a resource (e.g., the items and total price in a shopping cart).
*   **Transfer:** The movement of these resource representations between the client and server using standard HTTP methods.

---

## 4. URL Anatomy and Route Design Rules
A standardized URL has distinct parts: Scheme (`https`), Subdomain (`api`), Domain (`example.com`), Versioning (`v1`), and the Path/Resource (`/books`).

### Text Illustration: The API URL Structure
```text
  https://api.example.com/v1/organizations/123/projects?sort=desc
  \___/   \_____________/ \/ \___________/ \_/ \______/ \_______/
    │            │        │        │        │      │        │
  Scheme     Subdomain Version  Resource   ID   Resource  Query Params
```

### Golden Rules for Route Design
*   **Always use Plural Nouns:** The resource in a path segment must always be plural (e.g., `/organizations`, not `/organization`). Crucially, even when fetching a single entity, the route must remain plural (e.g., `/organizations/123` or `/books/123`) because the path segment represents the collection the resource belongs to.
*   **Formatting:** Do not use spaces, uppercase letters, or underscores in URLs. If a slug contains multiple words, convert it to lowercase and use hyphens (e.g., `/books/harry-potter`).
*   **Hierarchical Paths:** The forward slash (`/`) dictates a strict hierarchy. `/organizations/123/projects` clearly means "fetch the projects belonging to organization 123".

---

## 5. HTTP Methods and Idempotency
**Idempotency** means that performing an action multiple times yields the exact same side-effect on the server as performing it just once. 

*   **GET (Idempotent):** Fetches data. Does not alter the server state.
*   **PATCH (Idempotent):** Partially updates a resource (e.g., changing just a user's name). Repeatedly sending the same update results in the same final state.
*   **PUT (Idempotent):** Completely replaces a resource representation.
*   **DELETE (Idempotent):** Removes a resource. The first call deletes it; subsequent calls return a 404 error, but no *new* side effects are caused.
*   **POST (Non-Idempotent):** Creates a new resource. Calling it 1,000 times will generate 1,000 new distinct records with unique IDs.

### Text Illustration: Patch vs. Put
```text
Existing Entity: { "id": 1, "name": "John", "status": "active" }

PATCH Payload:   { "name": "Jane" } 
Result:          { "id": 1, "name": "Jane", "status": "active" } (Updates only name)

PUT Payload:     { "name": "Jane" } 
Result:          { "id": 1, "name": "Jane" } (Completely replaces, losing 'status')
```

---

## 6. The API Design Workflow
Do not write code immediately. API design should start with visual wireframes (like Figma) to understand how the end-user interacts with the platform. 

1.  **Extract Nouns:** Analyze wireframes and requirements to find nouns (e.g., Organization, Project, Task). These are your "Resources".
2.  **Design DB Schema:** Map out the tables based on these resources.
3.  **Identify Actions:** List the required CRUD (Create, Read, Update, Delete) operations for each resource.
4.  **Design the Interface First:** Use tools like Insomnia, Postman, or Swagger to map out the API routes, organize them into folders (e.g., `org` endpoints, `project` endpoints), and test payloads before writing server code. 

---

## 7. Standardizing CRUD Operations and Status Codes (From Demo)
*   **Create (POST):** The client payload should exclusively contain necessary fields (e.g., `name`, `description`) and omit server-generated fields like `ID` or `createdAt`. Returns **201 Created** along with the newly created resource data.
*   **List (GET):** Fetches multiple items (e.g., `GET /organizations`). Returns **200 OK**. **Crucial Rule:** If no items exist or a filter yields no results, it should return an empty array `[]` with a 200 code, *never* a 404.
*   **Fetch Single (GET):** Route is `/:resource/:id` (e.g., `GET /organizations/123`). Returns **200 OK**. If the ID does not exist (e.g., if it was previously deleted), return **404 Not Found**. 404 should *only* be used when a client specifically requests an exact resource ID that is missing.
*   **Update (PATCH):** Uses `/:resource/:id` (e.g., `PATCH /organizations/123`). Takes a partial payload (like just updating a status to "active") and returns **200 OK** with the updated data.
*   **Delete (DELETE):** Uses `/:resource/:id`. Returns **204 No Content** with an empty response body.

---

## 8. Designing Robust "List" APIs (Pagination, Sorting, Filtering)
When returning lists of data, APIs must implement Pagination, Sorting, and Filtering via query parameters to optimize server bandwidth and UI performance.

### Pagination
Uses `?limit=2&page=2`. The JSON payload must return a metadata object alongside the data array.

**Text Illustration: Paginated Response payload**
```json
{
  "data": [ { "id": 4, "name": "Org 4" }, { "id": 5, "name": "Org 5" } ],
  "total": 50,         <-- Total records in DB (e.g., total organizations)
  "page": 2,           <-- Current page requested
  "totalPages": 25     <-- Total pages available based on limit
}
```

### Sorting and Filtering
*   **Sorting:** Expose `?sortBy=name` and `?sortOrder=asc` parameters so the client can dynamically order data.
*   **Filtering:** Expose parameters like `?status=archived` or `?name=Org1` to strictly return matching entities in the list. 

---

## 9. Custom Actions (Non-CRUD)
Often, a client needs to perform an action that isn't a simple database update. For example, "archiving" an organization requires complex server logic like cascading deletions or notification emails.

*   **The Rule:** If an action does not fit into standard CRUD methods, it should be implemented as a **POST** request, appending the action verb to the end of the route.
*   **Demo Examples & Status Codes:** 
    *   `POST /organizations/123/archive`: Returns `200 OK` because it modifies the existing organization's state and performs background tasks.
    *   `POST /projects/123/clone`: Returns `201 Created` because the server action results in the generation of a brand new cloned project in the database.

---

## 10. Global Best Practices for Delightful APIs
*   **Consistent Payload Naming:** JSON fields must always use `camelCase`. You must keep payload keys identical across different resources. For instance, if you accept `description` when creating an organization, you must use `description` when creating a project.
*   **Sane Defaults:** Do not force clients to explicitly pass obvious parameters. 
    *   *List APIs:* If `page` is missing, default to `1`. If `limit` is missing, default to `10` or `20`. If `sortOrder` is missing, default to `descending` order of `createdAt`. 
    *   *Create APIs:* If creating a new organization and the client leaves the `status` field blank, the server should sensibly default it to `"active"`.
*   **Interactive Documentation:** Always provide an interactive documentation playground, such as Swagger or OpenAPI. This allows consumers to see consistent styling and test the APIs instantly without guesswork.
*   **Avoid Abbreviations:** Never use shorthand like `desc` for description. Write intuitive, fully readable fields because the consumer of the API lacks the context of the backend engineer who wrote it.

---

## 11. Supplemental "Good-to-Have" Points (External Context)
*Please note: The following concepts rely on standard industry knowledge outside of the provided source text.*

*   **HATEOAS (Hypermedia As The Engine Of Application State):** Mentioned as part of the "Uniform Interface" constraint. In a mature REST API, server responses include data and dynamic hyperlink URLs telling the client what actions they can perform next.
*   **UUIDs vs. Auto-Incrementing IDs:** While simple IDs like `1` or `2` are used for demos, modern APIs prefer **UUIDs** for public-facing URLs. Auto-incrementing IDs are a security risk as they allow attackers to guess IDs (Insecure Direct Object Reference or IDOR).
*   **Plural Noun Exceptions:** While the golden rule is to use plural nouns (`/users`), singletons (resources where only one can ever exist per user context) are an exception. A common example is `/profile` or `/me`, where pluralizing it wouldn't make semantic sense for the logged-in user.