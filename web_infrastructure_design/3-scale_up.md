# Web Infrastructure with Split Components

This README describes a web infrastructure hosting the website `www.foobar.com`. It highlights the difference between a **web server** and an **application server**, and explains why we split components into dedicated servers.

---

## Infrastructure Overview

### Components

1. **Load Balancer (HAProxy)**

   * One HAProxy instance, configured in a **cluster setup** with another load balancer.
   * Provides **high availability** and **traffic distribution** across web servers.
   * Prevents downtime in case one load balancer fails.

2. **Web Server (Nginx)**

   * Dedicated server running **Nginx**.
   * Handles **static content** (HTML, CSS, JavaScript, images).
   * Acts as the first entry point for HTTP/HTTPS requests.
   * Forwards dynamic requests to the application server.

3. **Application Server**

   * Dedicated server running the **application runtime** (Python/Flask, PHP-FPM, Node.js, etc.).
   * Processes **business logic**: user authentication, booking requests, API calls, form processing.
   * Interacts with the database when data storage or retrieval is needed.

4. **Database Server (MySQL)**

   * Dedicated server hosting **MySQL**.
   * Stores persistent data: user accounts, transactions, reviews, etc.
   * Centralized data ensures consistency across the infrastructure.

---

## Why Split Components

* **Separation of Concerns:**
  Each component (web, application, database) has its own role. This avoids resource contention and improves security and maintainability.

* **Scalability:**
  Web servers can be scaled horizontally (add more web servers) without touching the application or database layers.

* **Security:**
  Databases are isolated on their own server, accessible only by the application server. This reduces the risk of direct database exposure.

* **Performance Optimization:**

  * Web server is optimized for static file serving and SSL termination.
  * Application server is optimized for executing business logic.
  * Database server is optimized for queries and storage.

---

## Web Server vs Application Server

* **Web Server (Nginx/Apache):**

  * Primary role: serve **static files** and forward requests.
  * Lightweight, efficient in handling concurrent HTTP connections.
  * Terminates SSL (HTTPS) and may provide caching, compression, and load balancing.

* **Application Server (Gunicorn, uWSGI, PHP-FPM, Node.js runtime):**

  * Primary role: execute **dynamic content generation**.
  * Runs the actual application code (Python, PHP, Java, JavaScript, etc.).
  * Handles complex logic and database interactions.

---

## Why We Add Each Element

* **Extra Load Balancer (HAProxy Cluster):**
  Provides redundancy and avoids a single point of failure.

* **Dedicated Web Server:**
  Optimizes handling of HTTP requests and static content. Prevents static file serving from slowing down dynamic requests.

* **Dedicated Application Server:**
  Keeps business logic separate from static file serving, ensuring better performance and clearer scaling strategies.

* **Dedicated Database Server:**
  Isolates data storage and ensures only authorized connections. Improves performance by separating database load from web/app servers.

---

## High-Level Diagram

```
          User
           |
           v
    DNS -> www.foobar.com
           |
    +--------------------+
    |  Load Balancer(s)  |
    |  (HAProxy cluster) |
    +--------------------+
           |
   -------------------------
   |                       |
+---------+          +----------------+
| Web     |          | Application    |
| Server  | <------> | Server         |
| (Nginx) |          | (Flask/PHP/etc)|
+---------+          +----------------+
                           |
                      +-----------+
                      | Database  |
                      | (MySQL)   |
                      +-----------+
```

---

✅ This architecture is more robust than a single-server setup. It separates responsibilities, improves security, and allows scaling each layer independently.
