## **Step 1: User Access**

A user types `www.foobar.com` in the browser.
The DNS system resolves `www` to the IP address of the **load balancer**.

---

## **Step 2: Load Balancer (HAProxy)**

* **Why we add it:**
  The load balancer distributes incoming traffic across multiple servers to avoid overloading a single machine.
* **Algorithm:**
  Configured with **Round Robin** distribution (the most common one). It works by sending the first request to Server A, the second request to Server B, the third to Server A again, and so on.
* **Active-Active vs Active-Passive:**

  * **Active-Active:** Both servers handle requests at the same time. This is the case here, since both backend servers are active and the load balancer distributes requests across them.
  * **Active-Passive:** Only one server is active; the other is on standby and takes over only if the active server fails.

---

## **Step 3: Backend Servers**

We now have **two servers** behind the load balancer, each containing:

1. **Web Server (Nginx):**

   * Handles incoming HTTP/HTTPS requests.
   * Serves static files.
   * Forwards dynamic requests to the application server.

2. **Application Server:**

   * Runs the website’s logic (authentication, payment, etc.).
   * Processes requests and interacts with the database.

3. **Application Files (Code Base):**

   * The website’s source code, deployed on both servers.
   * This ensures that whichever server handles the request, the application responds the same way.

4. **Database (MySQL, with Primary-Replica configuration):**

   * This is not duplicated on both servers independently, but rather organized as a **cluster**.

---

## **Step 4: Database Cluster (Primary-Replica)**

* **How it works:**

  * The **Primary (Master) node** handles **all write operations** (INSERT, UPDATE, DELETE).
  * The **Replica (Slave) nodes** copy data asynchronously from the Primary and handle **read-only queries** (SELECT).
* **Difference in usage by the application:**

  * The application server sends all **writes** to the Primary.
  * The application server can send **reads** either to the Primary or to the Replicas, depending on configuration.
  * This improves performance and availability because heavy read traffic is spread across multiple nodes.

---

## **Whiteboard Diagram**

```
User --> DNS --> Load Balancer (HAProxy)
                        |
          -------------------------------
          |                             |
     Server A                       Server B
  (Nginx + App + Code)         (Nginx + App + Code)
          |                             |
          -------------------------------
                        |
                MySQL Database Cluster
                (Primary <-> Replica)
```

---

## **Step 5: Issues with this Infrastructure**

1. **Single Point of Failure (SPOF):**

   * The **load balancer** itself is a SPOF. If it fails, the entire website is unavailable.
   * The **database Primary** is also a SPOF for writes. If it fails, no new data can be written.

2. **Security Issues:**

   * No **firewall** to filter malicious traffic.
   * No **HTTPS** means communication is unencrypted, exposing sensitive data (e.g., passwords).

3. **No Monitoring:**

   * There is no system in place to track performance, detect failures, or alert administrators.
   * Without monitoring, downtime or overload can go unnoticed until users complain.
