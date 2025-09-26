### **Step 1: The User Accesses the Website**

A user opens their browser and types `www.foobar.com`.
This starts a chain of actions that allow the user to reach the hosted website.

---

### **Step 2: Domain Name Resolution**

* The domain name is `foobar.com`.
* The `www` part is a **DNS record** (specifically a **CNAME** or an **A record**) that maps to the server’s IP address (`8.8.8.8`).
* DNS ensures that humans use readable names like `foobar.com`, while machines communicate with IP addresses.

---

### **Step 3: The Server**

* A **server** is a physical or virtual machine that provides services (like web hosting) to clients.
* In this case, it is **one single server** hosting the entire stack.

The server contains:

1. **Web Server (Nginx):**

   * Handles HTTP/HTTPS requests.
   * Serves static files (like HTML, CSS, JavaScript, images).
   * Passes dynamic requests to the application server.

2. **Application Server:**

   * Executes the application code (for example, a Python, PHP, or Node.js runtime).
   * Processes the logic of the website (authentication, form submissions, etc.).
   * Interacts with the database when needed.

3. **Application Files (Code Base):**

   * The source code of the website (business logic, templates, assets).
   * Stored on the server and executed by the application server.

4. **Database (MySQL):**

   * Stores persistent data: users, posts, bookings, etc.
   * Responds to queries from the application server.

---

### **Step 4: Communication with the User**

* The server communicates with the user’s computer via the **HTTP/HTTPS protocol** over the internet.
* This allows browsers to render the website properly.

---

### **Whiteboard Diagram (High-Level)**

```
User ---> DNS ---> Server (8.8.8.8)
                     |
                     |-- Nginx (Web server)
                     |      |
                     |      |-- Application server
                     |              |
                     |              |-- Application files (code base)
                     |              |-- MySQL Database
```

---

### **Step 5: Issues with This Infrastructure**

1. **Single Point of Failure (SPOF):**

   * Only one server. If it crashes, the entire website becomes unavailable.

2. **Downtime During Maintenance:**

   * Updating the code or restarting Nginx will cause the website to be temporarily unreachable.

3. **Scalability Problems:**

   * If traffic increases significantly, one server cannot handle too many requests at the same time.
   * There is no load balancing or redundancy.
