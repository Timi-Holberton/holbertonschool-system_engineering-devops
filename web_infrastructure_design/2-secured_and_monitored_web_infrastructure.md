# Whiteboard (ASCII) — high level

```
                                   Internet
                                      |
                                  Firewall #1
                                      |
                                DNS -> www.foobar.com
                                      |
                                  Load Balancer
                                 (HAProxy w/ SSL)
                                      |
                      -----------------------------------
                      |                                 |
                 Firewall #2                         Firewall #3
                      |                                 |
                 App Server A                      App Server B
        (Nginx + App server + App files)     (Nginx + App server + App files)
            |  Monitoring client A                 |  Monitoring client B
            |                                       |
            |-----------\                     /-----|
                        |                   |
                  Private network (VLAN) / backend
                        |                   |
                        |               MySQL Server
                        |      (Primary for writes; Replica(s) could exist)
                        |               Monitoring client C
                        |
                 (optional: internal load-balancer health checks)
```

Notes:

* The *three servers* in the diagram are: **App Server A**, **App Server B**, and **MySQL Server**.
* The HAProxy load balancer is a separate appliance/service in front of them (not counted among the three servers).
* Firewalls #1,#2,#3 are placed at different layers (perimeter and segment-level).
* Each server runs a **monitoring client** (collector/agent) that sends logs/metrics to a remote monitoring service (SumoLogic, Datadog, Prometheus+Pushgateway, etc.).
* A single SSL certificate for `www.foobar.com` is installed on the load balancer for HTTPS (TLS) termination (we’ll discuss trade-offs below).

---

# Why each additional element is added

**Perimeter Firewall (#1)**

* Blocks unwanted traffic at the edge (deny-by-default, allow only relevant ports like 443 & 80 to the LB).
* Protects the whole network from simple scans, brute force, and other broad attacks.

**Firewall #2 & Firewall #3 (segment / host-level)**

* Firewall #2 protects App Server A, Firewall #3 protects App Server B (or they can be host-based firewalls).
* They enforce stricter rules (only allow ports: 80/443 from LB, SSH from admin IPs, and DB ports from the internal network).
* Adds defense-in-depth: even if perimeter firewall is bypassed, host firewalls limit lateral movement.

**SSL certificate for `www.foobar.com`**

* Ensures encrypted traffic between users’ browsers and our infrastructure (confidentiality, integrity, authentication).
* Required for modern browsers and security best practices.

**Monitoring clients (3)**

* Each server runs an agent (SumoLogic collector, Datadog Agent, Prometheus exporter + push or node_exporter + Prometheus pull, etc.).
* Agents collect logs, metrics, system telemetry (CPU, memory), and app metrics and push them securely to the monitoring backend.
* Gives observability: alerts, dashboards, forensic logs.

---

# What firewalls are for (short)

* **Filter traffic** by source/destination/port/protocol.
* **Block/allow** only required services (e.g., allow 443 to LB, allow internal DB port only from the app subnet).
* **Reduce attack surface** and contain compromises (isolate hosts).

---

# Why serve traffic over HTTPS

* **Encryption:** protects user data (passwords, tokens) from eavesdropping.
* **Integrity:** prevents man-in-the-middle alteration of content.
* **Authentication & trust:** browsers show the lock icon and trust the site.
* **Required by modern standards:** many browser features (geolocation, service workers) require HTTPS.

---

# What monitoring is used for

Monitoring provides:

* **Metrics**: CPU, memory, disk, network, request rates, error rates, latency.
* **Logs**: access logs, error logs, application logs for debugging.
* **Alerts**: notify operators when thresholds are crossed (e.g., high error rate, low disk).
* **Dashboards & trends**: analyze historical behavior and capacity planning.
* **Tracing** (optionally): track requests across services for performance issues.

---

# How the monitoring tool collects data

Typical model used here:

1. **Agent-based, push**: A lightweight agent runs on App A, App B and the MySQL server. The agent:

   * reads system metrics (CPU, memory),
   * tails application and Nginx logs,
   * scrapes Nginx or app metrics endpoints (e.g., `/status` or Prometheus exporter),
   * pushes data securely (TLS) to the remote monitoring provider (SumoLogic HTTPS ingestion endpoint or Datadog API).
2. **Secure transport**: agents authenticate with API keys and send compressed/encrypted payloads to the vendor over HTTPS.
3. **Backend processing**: the monitoring service indexes metrics/logs, generates dashboards, and fires alerts.

(Alternative: Prometheus pull model: a central Prometheus server scrapes endpoints exposed by the servers. This requires the Prometheus server to have network access to the targets and typically runs in the monitoring stack.)

---

# How to monitor web server QPS (Queries / Requests Per Second)

Steps (practical):

1. **Enable Nginx metrics**:

   * Use the **nginx_stub_status** module (simple metrics) or install an **nginx prometheus exporter** to expose request rates, active connections, etc.
2. **Collect access logs**:

   * Parse Nginx access logs (for example using the monitoring agent or a log-forwarder) to count requests per second by time window.
3. **Agent config**:

   * Configure monitoring agent to scrape the exporter endpoint (e.g., `http://localhost:9113/metrics`) at a 10s interval.
4. **Dashboards & alerts**:

   * Create a dashboard that shows `rate(nginx_http_requests_total[1m])` or equivalent metric for QPS.
   * Alert on unexpected drops or spikes (e.g., QPS > X for 5 minutes or QPS = 0 for 2 minutes).
5. **Optional synthetic checks**:

   * Add an external uptime/synthetic probe that hits `/health` every minute; use that in conjunction with QPS metrics for more robust detection.

---

# How the monitoring data flows (example)

```
[App Server A]  -> agent -> TLS -> SumoLogic ingestion endpoint
[App Server B]  -> agent -> TLS -> SumoLogic ingestion endpoint
[MySQL Server]  -> agent -> TLS -> SumoLogic ingestion endpoint
```

Agent may also forward logs to a centralized log indexer (ELK/Opensearch) in the private network before shipping to SaaS.

---

# Issues & trade-offs with this setup (honest view)

### 1) Why terminating SSL at the load balancer can be an issue

* **Pros** of terminating at LB: easier certificate management (single place to renew), offloads CPU work from app servers, lets LB inspect HTTP headers for routing.
* **Cons / issues**:

  * **Traffic inside the internal network becomes plaintext** (unless you re-encrypt to backends). If an attacker or misconfiguration exposes internal network traffic, sensitive data could leak.
  * **End-to-end encryption lost**: compliance requirements (e.g., some PCI/DSS or health data rules) may require end-to-end TLS.
  * **Authentication/Audit complications**: if you rely on client certs or mutual TLS, terminating at LB requires extra configuration to forward client cert info securely to backends.
* **Mitigation**: use TLS passthrough on the LB (so backends handle TLS) or use **re-encryption**: LB terminates TLS then opens another TLS connection to the backend (mutual TLS between LB and backends).

### 2) Why having only one MySQL server accepting writes is an issue

* **Single writer is a SPOF for writes**: if the primary fails, writes stop. Application behavior during failover must be handled (promote a replica to primary).
* **Failover complexity**: promoting a replica and ensuring no split-brain requires orchestration (manual or automated with tools like MHA, Orchestrator, Patroni for PostgreSQL).
* **Replication lag**: replicas may lag behind the primary, causing stale reads if read-scaling is used carelessly.
* **Mitigation**: use automatic failover, multi-primary clustering (careful), or manage a highly available DB cluster (Galera, Group Replication) depending on write patterns and consistency needs.

### 3) Why having servers that all contain the same components (DB + app + web on same hosts) might be a problem

* **Resource contention**: web/app workloads can compete with DB for CPU, memory, and disk I/O—leading to unpredictable performance.
* **Security & isolation**: a compromise of the app host can expose the database if co-located, increasing blast radius.
* **Scalability**: scaling the web/app tier independently of the DB tier is difficult if they’re bundled together. You’d waste resources or over-provision components that don’t need scaling.
* **Operational complexity**: patching/maintenance windows affect multiple roles at once; upgrading the DB becomes tangled with app deployments.
* **Mitigation**: separate roles: dedicated DB hosts (or managed DB service), dedicated stateless app servers, and use shared storage/replication for persistence.

---

# Remaining single points of failure in this design

* **Single load balancer** — if HAProxy fails, site is down. Mitigate by adding a second LB with VRRP (keepalived) for active-passive or using DNS-based failover for geo/active-active.
* **Single MySQL primary** — writes stop if primary fails. Use replicas + automated failover or managed DB with HA.
* **Perimeter firewall device** — consider redundant firewall appliances or cloud security groups.

---

# Short summary / action items to reach production readiness

1. **Add LB redundancy**: HAProxy + keepalived (active-passive) or pair of LBs in active-active with consistent hashing if needed.
2. **Encrypt internal traffic** or re-encrypt at LB -> backend using mTLS.
3. **Harden host firewalls** and limit SSH to trusted IPs (use jump host + 2FA).
4. **Separate DB onto dedicated host(s)** and add replica(s) + automated failover.
5. **Centralized monitoring & alerting** with collectors on every host; add dashboards for QPS, latency, errors.
6. **Add intrusion detection, WAF and backups** (DB backups and backup verification).

