# Lab: Apache HTTP Server Denial-of-Service via Slowloris (MPM Comparison)

The lab demonstrates a low-bandwidth Layer 7 Denial-of-Service (Slowloris) attack against Apache HTTP Server 2.4. This lab highlights how different Multi-Processing Modules (`event`, `worker`, and `prefork`) manage connection states, why thread/process pool exhaustion occurs despite asynchronous event loops, and how to properly remediate it.

---

## 1. Technical Overview

### Vulnerability Mechanism
Slowloris exploits the way web servers handle incomplete HTTP requests. By initiating valid TCP connections and sending partial HTTP headers at slow, periodic intervals (without terminating with `\r\n\r\n`), the attacker ties up active worker threads/processes indefinitely.

### Resource
For demonstration purposes, all MPM modules are constrained to a deterministic ceiling:
* **Target constraint:** `MaxRequestWorkers = 30`
* **Exploit footprint:** A payload generating **33–40 slow sockets** (> `MaxRequestWorkers`).

### MPM behavior
* **`mpm_prefork` (Process-based):** Each socket holds an entire process. Once 30 sockets are opened, 100% of child processes are locked in the `Reading Request` state.
* **`mpm_worker` (Multi-thread, synchronous):** Sockets consume worker threads directly until all 30 threads across the server pool are exhausted.
* **`mpm_event` (Asynchronous event-driven):** Even though mpm_event offloads idle Keep-Alive connections, reading partial headers still locks active worker threads. Opening 30 slow connections drives idle_workers down to 0 and exhaust the server.

---

## 2. Key Configuration Directives (`httpd.conf` & `extra/httpd-mpm.conf`)

1. **Activate Custom Constraints:**
   ```apache
   Include conf/extra/httpd-mpm.conf
   ```

2. **Disable Defensive Rate-Limiting:**
   ```apache
   # LoadModule reqtimeout_module modules/mod_reqtimeout.so
   ```

---

## 3. Execution

### Step 1: Deploy The Environment
Spawn a MPM profile and automatically starts the exploit:

```bash
# Option A: Event MPM (Asynchronous)
docker compose --profile event up -d

# Option B: Worker MPM (Multi-threaded)
docker compose --profile worker up -d

# Option C: Prefork MPM (Process-isolated)
docker compose --profile prefork up -d
```

### Step 2: Verify Active MPM Engine
Ensure the container runs the targeted engine:

```powershell
docker exec -it victim-server httpd -V | Select-String "Server MPM"
```

### Step 3: Verify DoS
Verify denial of service from the host machine using cURL:

```powershell
curl.exe -I -m 5 -s -o /dev/null -w "HTTP_CODE: %{http_code}\nTIME_TOTAL: %{time_total}s\n" http://localhost:8080/
```

**Output:**
```
HTTP_CODE: 000
TIME_TOTAL: 5.007431s
```
*(HTTP Code `000` and execution hitting the strict 5-second boundary indicates zero workers are available to process the incoming handshake).*

---


## 4. Mitigation & Defensive Hardening

To harden Apache against Slowloris-style resource exhaustion:

1. **Enable and Configure `mod_reqtimeout`:**
   Enforce minimum transfer rates and read timeouts for request headers and bodies:
   ```apache
   LoadModule reqtimeout_module modules/mod_reqtimeout.so

   <IfModule reqtimeout_module>
       # Allow 10s to receive header, then require at least 500 bytes/sec
       RequestReadTimeout header=10-20,MinRate=500 body=10,MinRate=500
   </IfModule>
   ```

2. **Reverse Proxy Offloading:**
   Place an event-driven reverse proxy (e.g., NGINX, HAProxy) in front of Apache to buffer slow headers completely before forwarding the request to backend workers.

3. **Connection Limits per IP:**
   Utilize iptables or modules like `mod_qos` / `mod_evasive` to limit maximum concurrent connections per source IP address.


## 5. Credits
* **Slowloris exploit PoC:** The Python payload script used in this demonstration is attributed to **gkbrk** (available at https://github.com/gkbrk/slowloris). 
