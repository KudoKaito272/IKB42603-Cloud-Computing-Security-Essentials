# Lab 4: Access Control and Network Security

## Student Information
- **Student Name:** Muhammad Afiq Farhan bin Mohd Nasaruddin
- **Course Code:** IKB42603 Cloud Computing Security Essentials
- **Lab Title:** Lab 4 - Access Control and Network Security
- **Lecturer:** Nor Adani Kamal Mohamad Nasir

---

**Topic:** Authentication, MFA, RBAC, network segmentation, firewall policy and container hardening  
**Tools:** Docker, Nginx, curl, oathtool, kind, kubectl, iptables and Trivy

## Lab Learning Outcomes

At the end of this lab, I was able to:

1. Distinguish authentication from authorization and enforce both controls.
2. Add and validate a time-based one-time password as a second factor.
3. Apply Kubernetes RBAC using a least-privilege developer role.
4. Segment frontend, backend and database services with Docker networks.
5. Apply a default-deny firewall policy with explicit allow rules.
6. Harden a container using a non-root user, read-only storage, dropped capabilities and `no-new-privileges`.
7. Scan a container image for high and critical vulnerabilities.

## Course and Assessment Mapping

| Item | Mapping |
|------|---------|
| **Course Learning Outcome** | CLO2 - Construct secure cloud operations that safeguard data integrity |
| **Lecture Topics** | Week 5 (Access Control) and Week 9 (Network Security patterns) |
| **Value/Skill Clusters** | VBE3 (Integrity) and SC8 (Integrated Problem-Solving) |
| **Assessment** | Lab report with screenshots, command output and short answers |

## Lab Arrangement

| Session | Week | Focus |
|---------|------|-------|
| **Session A** | Week 5 | Authentication, MFA and RBAC enforcement (Tasks 1-3) |
| **Session B** | Week 5 | Network segmentation, firewall rules and container hardening (Tasks 4-6) |

**Note:** Session A controls who can enter and what an identity may do. Session B controls which services can communicate and reduces the damage possible after a compromise.

## Introduction

This laboratory demonstrated layered access and network security controls in a container and Kubernetes environment. HTTP Basic Authentication showed that unauthenticated requests are rejected. TOTP added a second factor, while Kubernetes RBAC restricted a developer service account to reading pods. Docker network segmentation then separated the internet-facing web tier from the database tier. Finally, a default-deny firewall and a hardened non-root container reduced unnecessary network and host privileges.

The evidence images in this report should be replaced with screenshots captured from the terminal after each task. Each placeholder identifies the exact output that should be visible.

## Environment and Preparation

The commands were run from the Lab 4 directory using Docker, kind, kubectl, oathtool and Trivy. The Kubernetes cluster was named `ccse-lab4`, and the application namespace was `app`.

Before starting, Docker was running and the required tools were available:

```bash
docker --version
kind --version
kubectl version --client
oathtool --version
trivy --version
```

## Task 1: Authentication with HTTP Basic Authentication

### Procedure

An Apache htpasswd file was generated for the test user `student`. Nginx was configured to require that file before returning the response:

```bash
docker run --rm httpd:alpine htpasswd -nbB student 'P@ssw0rd!' > htpasswd.txt

cat > default.conf <<'EOF'
server { listen 80;
	location / { auth_basic "Restricted";
		auth_basic_user_file /etc/nginx/.htpasswd;
		return 200 'Authenticated OK\n'; } }
EOF

docker run --rm -d --name authsvc -p 8080:80 \
	-v "$(pwd)/default.conf:/etc/nginx/conf.d/default.conf:ro" \
	-v "$(pwd)/htpasswd.txt:/etc/nginx/.htpasswd:ro" nginx

curl -s -o /dev/null -w 'no-creds: %{http_code}\n' http://localhost:8080
curl -s -u student:'P@ssw0rd!' http://localhost:8080
```

### Result and observation

The request without credentials returned `no-creds: 401`. The request with the correct username and password returned HTTP 200 and `Authenticated OK`. This demonstrates **authentication (AuthN)**: the service verifies the identity's credentials before allowing access. The password file stores a bcrypt hash rather than the plaintext password.

### Evidence

![Task 1: HTTP Basic Authentication showing 401 and 200](img/1.3.png)

**Evidence placeholder:** Paste a screenshot containing both `no-creds: 401` and the successful `Authenticated OK` response.

## Task 2: Multi-Factor Authentication with TOTP

### Procedure

A random Base32 secret was generated and used to calculate the current six-digit TOTP code:

```bash
SECRET=$(head -c20 /dev/urandom | base32)
echo "Enrol this secret in an authenticator app: $SECRET"
oathtool --totp -b "$SECRET"

read "CODE?Enter the 6-digit code: "
[ "$CODE" = "$(oathtool --totp -b "$SECRET")" ] && echo 'MFA OK' || echo 'MFA FAILED'
```

The generated secret can be enrolled in an authenticator application. The code changes periodically, so it should be entered before it expires.

### Result and observation

Entering the current TOTP value produced `MFA OK`. TOTP combines something the user knows, such as a password, with something the user has, such as an enrolled authenticator device. A stolen password alone is therefore insufficient to pass the second verification step.

### Evidence

![Task 2: TOTP generation and MFA validation](img/2.png)

**Evidence placeholder:** Paste a screenshot showing the generated TOTP test and the final `MFA OK` result. Do not include a reusable real account secret in the submitted report.

## Task 3: Authorization with Kubernetes RBAC

### Procedure

A kind cluster, namespace and developer service account were created. The role allows only `get` and `list` operations on pods:

```bash
kind create cluster --name ccse-lab4
kubectl create namespace app
kubectl create serviceaccount dev -n app

kubectl create role dev-role -n app --verb=get,list --resource=pods
kubectl create rolebinding dev-rb -n app --role=dev-role --serviceaccount=app:dev

SA=system:serviceaccount:app:dev
kubectl auth can-i list pods -n app --as=$SA
kubectl auth can-i create deploy -n app --as=$SA
kubectl auth can-i delete pods -n app --as=$SA
```

### Result and observation

The expected results were:

```text
yes
no
no
```

The developer service account could list pods but could not create deployments or delete pods. This demonstrates **authorization (AuthZ)**: after an identity is known, RBAC determines which actions that identity is permitted to perform. The role binding is namespace-scoped, so it does not grant cluster-wide permissions.

### Evidence

![Task 3: SetUp](img/3.1.png)
![Task 3: Kubernetes RBAC can-i results](img/3.2.png)

**Evidence placeholder:** Paste a screenshot showing the service account and all three `kubectl auth can-i` results.

## Task 4: Three-Tier Network Segmentation

### Procedure

Two Docker networks were created. The database was attached only to the backend network, the application to both networks, and the web tier only to the frontend network:

```bash
docker network create frontend-net
docker network create backend-net

docker run -d --name db  --network backend-net  redis:alpine
docker run -d --name app --network backend-net  nginx
docker network connect frontend-net app
docker run -d --name web --network frontend-net nginx

docker exec web sh -c 'apt-get update -qq && apt-get install -y -qq curl && curl -s -m 3 db:6379 || echo BLOCKED'
docker exec app sh -c 'apt-get update -qq && apt-get install -y -qq netcat-openbsd && nc -z -w3 db 6379 && echo REACHABLE'
```

### Result and observation

The web-to-database test returned `BLOCKED`, while the application-to-database test returned `REACHABLE`. The frontend container has no network path to the database because it is not attached to `backend-net`. The application is the controlled middle tier and is the only service connected to both segments. This limits lateral movement if the internet-facing web container is compromised.

### Evidence

![Task 4: SetUp](img/4.1.png)
![Task 4: Network segmentation showing BLOCKED and REACHABLE](img/4.2.png)

**Evidence placeholder:** Paste a screenshot showing `BLOCKED` for `web -> db` and `REACHABLE` for `app -> db`.

## Task 5: Default-Deny Firewall Rules

### Procedure

The following throwaway container models a host firewall. Its default input policy is DROP, with only HTTPS and loopback explicitly allowed:

```bash
docker run --rm --cap-add=NET_ADMIN alpine sh -c '\
	apk add -q iptables; \
	iptables -P INPUT DROP; \
	iptables -A INPUT -p tcp --dport 443 -j ACCEPT; \
	iptables -A INPUT -i lo -j ACCEPT; \
	iptables -L INPUT -n'
```

### Result and observation

The ruleset showed `policy DROP` for `INPUT`, followed by explicit ACCEPT rules for TCP port 443 and the loopback interface. All other inbound traffic was denied by default. This is the same least-privilege model used by cloud security groups: traffic is permitted only when a matching rule explicitly allows it.

### Evidence

![Task 5: iptables default-deny policy](img/5.png)

**Evidence placeholder:** Paste a screenshot of `iptables -L INPUT -n` showing the DROP policy and the two ACCEPT rules.

## Task 6: Container and Host Hardening

### Procedure

A hardened Nginx image was started with a non-root identity, a read-only root filesystem, no Linux capabilities and the `no-new-privileges` security option. A temporary `/tmp` filesystem was supplied for applications that need temporary writes:

```bash
docker run -d --name hardened \
	--user 1000:1000 \
	--read-only \
	--cap-drop=ALL \
	--security-opt no-new-privileges \
	--tmpfs /tmp \
	nginxinc/nginx-unprivileged

docker inspect hardened --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}}'
docker inspect hardened --format '{{json .HostConfig.CapDrop}}'

docker run --rm aquasec/trivy image --severity HIGH,CRITICAL nginx:alpine | head -20
```

### Result and observation

The inspect output confirmed a non-root user and `ReadOnly=true`; the capability output confirmed that all capabilities were dropped. The Trivy command produced the image vulnerability summary for HIGH and CRITICAL findings. The scan result should be recorded with its actual count because vulnerability databases change over time.

Three important hardening measures and the attacks they blunt are:

| Measure | Security effect | Attack reduced |
|---------|-----------------|----------------|
| Non-root user | Removes unnecessary root privileges | Container escape or service compromise becoming full root on the host |
| `--cap-drop=ALL` | Removes privileged Linux capabilities | Network, filesystem or kernel abuse requiring extra capabilities |
| `--read-only` | Prevents modification of the root filesystem | Malware persistence and unauthorised alteration of binaries or configuration |
| `no-new-privileges` | Prevents processes gaining additional privileges | Privilege escalation through setuid or file capabilities |
| Minimal scanned image | Reduces packages and identifies vulnerable components | Exploitation of unnecessary or unpatched software |

### Evidence

![Task 6.1: SetUp](img/6.1.png)

![Task 6.2: Hardened container inspect output](img/6.2.png)

![Task 6.3: Trivy HIGH and CRITICAL scan summary](img/6.3.png)

**Evidence placeholders:** Paste one screenshot showing `User`, `ReadOnly` and `CapDrop`, and another showing the Trivy image name and HIGH/CRITICAL summary.

## Verification

The final RBAC and container settings were verified with:

```bash
kubectl get rolebinding dev-rb -n app -o yaml
docker inspect hardened --format '{{json .HostConfig.CapDrop}}'
```

The role binding YAML should identify `dev-role`, namespace `app` and service account `dev`. The container command should return a list containing `ALL`.

![Verification: RBAC role binding and capability drop](img/verify.png)

**Evidence placeholder:** Paste the verification screenshot here.

## Short-Answer Questions

### Q1. Explain the difference between authentication and authorization using Tasks 1 and 3.

Authentication verifies **who** a subject is. In Task 1, Nginx checked the username and password and returned HTTP 401 without valid credentials. Authorization determines **what** an authenticated subject may do. In Task 3, Kubernetes identified the `dev` service account and applied its RBAC role. The account could list pods but was denied permission to create deployments or delete pods. Authentication comes before authorization, but successful authentication does not automatically grant permission to every operation.

### Q2. Why is MFA so effective, and which attacks does it defeat?

MFA requires independent factors from different categories. Here, the password is something the user knows and the TOTP code is generated by something the user has. A stolen or guessed password is therefore not enough. MFA is effective against password spraying, credential stuffing, brute-force password reuse and many phishing incidents where only the password is captured. It does not stop real-time phishing proxies, malware on the authenticator device, stolen recovery codes or SIM-swapping. Phishing-resistant hardware security keys are stronger than ordinary OTP for high-risk accounts.

### Q3. How does network segmentation limit the damage of a compromised web server?

Segmentation removes unnecessary communication paths. In Task 4, `web` is connected only to `frontend-net`, while `db` is connected only to `backend-net`; therefore `web` cannot connect to the database through the Docker network. A compromised web server cannot immediately read or attack the database directly. The application tier remains a controlled intermediary. Segmentation does not replace application authentication or database authorization, but it reduces lateral movement and limits the blast radius.

### Q4. What does a default-deny firewall policy achieve, and how does it relate to cloud security groups?

A default-deny policy rejects traffic unless an explicit rule permits it. In Task 5, the INPUT chain dropped traffic by default and allowed only HTTPS on port 443 and loopback traffic. This prevents accidentally exposed ports and follows least privilege. Cloud security groups use the same general model: administrators define source, destination, protocol and port rules, while traffic without a matching allow rule is denied. An overly broad allow rule can still undermine default deny.

### Q5. List the hardening measures you applied and the attack surface each one removes.

The container ran as UID/GID `1000:1000`, so a service compromise did not begin with root privileges. All Linux capabilities were dropped, removing privileged operations that were not needed. The root filesystem was read-only, preventing persistence through changes to installed binaries and configuration. `no-new-privileges` blocked attempts to gain extra privileges through setuid or file capabilities. A temporary filesystem was limited to `/tmp`, preserving necessary temporary-write behaviour without making the whole root filesystem writable. Trivy scanned the image so known HIGH and CRITICAL vulnerabilities could be identified and remediated through image updates or replacement.

---

# Expansion Ideas

The following sections complete the advanced ideas from the lab manual. Each section includes an implementation approach and an evidence slot to fill after the extension is run.

## Expansion 1 - Web Application Firewall with ModSecurity and OWASP Core Rule Set

**Objective:** Deploy a ModSecurity WAF with the OWASP Core Rule Set (CRS) in front of an Nginx backend to block SQL injection and other common web application attacks.

#### Architecture Overview

The WAF deployment follows this architecture:

```
┌─────────────────────┐
│    Kali Linux       │
│   (curl client)     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────┐
│  ModSecurity + CRS      │
│  (WAF on port 8082)     │
└──────────┬──────────────┘
           │
      ┌────┴─────┐
      │          │
   Normal    Suspicious
      │          │
      ▼          ▼
┌───────────┐  BLOCK
│  Nginx    │  (403)
│ Backend   │
└───────────┘
  (200 OK)
```

The WAF sits between the client and the backend Nginx service. Both services run in a shared Docker network. The WAF inspects all requests using ModSecurity and the OWASP CRS. Legitimate requests are forwarded to the backend; suspicious requests are blocked with HTTP 403.

#### Step 2 — Create a dedicated WAF working directory

Create a project directory and navigate into it:

```bash
mkdir -p ~/waf-lab
cd ~/waf-lab
pwd
```

Expected output:
```
/home/paan/waf-lab
```

Note: Your username and path may differ based on your system configuration.

#### Step 3 — Create the Docker Compose configuration

Create a `docker-compose.yml` file to define both the backend and WAF services:

```bash
nano docker-compose.yml
```

Paste the following configuration:

```yaml
services:

  backend:
    image: nginx:alpine
    container_name: waf-backend
    networks:
      - waf-net

  waf:
    image: owasp/modsecurity-crs:nginx
    container_name: waf
    ports:
      - "8082:8080"
    environment:
      SERVERNAME: localhost
      BACKEND: http://backend:80
      BLOCKING_PARANOIA: 1
      ANOMALY_INBOUND: 5
      ANOMALY_OUTBOUND: 4
    depends_on:
      - backend
    networks:
      - waf-net

networks:
  waf-net:
```

Save the file with `Ctrl + O`, press `Enter`, and then exit with `Ctrl + X`.

Verify the file was created correctly:

```bash
cat docker-compose.yml
```

#### Step 4 — Verify the Docker Compose configuration

Confirm the file contains the correct services and network:

```bash
cat docker-compose.yml
```

Expected output should show:
- `backend` service using `nginx:alpine`
- `waf` service using `owasp/modsecurity-crs:nginx`
- Port mapping `8082:8080` for the WAF
- Environment variables including `BACKEND=http://backend:80`
- A shared `waf-net` network

#### Step 5 — Start the WAF and backend containers

Launch both services in the background:

```bash
docker compose up -d
```

Expected output:
```
[+] Running 3/3
 ✔ Network waf-lab_waf-net    Created
 ✔ Container waf-backend       Started
 ✔ Container waf               Started
```

#### Step 6 — Verify the containers are running

Check that both containers started successfully:

```bash
docker ps
```

Expected output should show:
- Container `waf` listening on port `0.0.0.0:8082->8080/tcp`
- Container `waf-backend` with status `Up`

#### Step 7 — Test backend connectivity directly

Before testing through the WAF, verify that the Nginx backend is responsive:

```bash
docker exec waf-backend wget -qO- http://localhost
```

Expected output should be the default Nginx welcome page:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</head>
...
</html>
```

This confirms the backend service is running and responding to HTTP requests.

#### Step 8 — Verify WAF-to-backend DNS resolution

Ensure the WAF container can resolve the backend's Docker DNS name:

```bash
docker exec waf getent hosts backend
```

Expected output:
```
172.x.x.x    backend
```

This proves the WAF can communicate with the backend through the Docker network. DNS resolution is critical for avoiding 502 Bad Gateway errors.

Now check the WAF logs for startup messages:

```bash
docker logs waf
```

Look for messages indicating that Nginx and ModSecurity started successfully.

#### Step 9 — Test legitimate HTTP traffic through the WAF

Send a normal HTTP request to the WAF:

```bash
curl -i http://localhost:8082/
```

Expected output:
```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 612
...
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

This demonstrates that:
- curl successfully connects to the WAF on port 8082
- The WAF accepts the legitimate request
- The WAF forwards the request to the backend
- The backend responds with HTTP 200
- The WAF relays the response back to the client

#### Step 10 — Test the WAF with a malicious SQL injection request

Send a request containing a harmless pattern that resembles SQL injection:

```bash
curl -i "http://localhost:8082/?id=1%20OR%201%3D1"
```

The decoded request is equivalent to:
```
GET /?id=1 OR 1=1
```

Expected output:
```
HTTP/1.1 403 Forbidden
Content-Type: text/html
Content-Length: xxx
...
```

The WAF inspects the request, identifies the SQL injection pattern, and blocks it with HTTP 403 Forbidden. The exact response content may vary depending on the CRS version, but the HTTP status should be 403.

#### Step 11 — Examine the WAF logs for rule detection

Check the WAF logs to confirm ModSecurity detected and blocked the request:

```bash
docker logs waf 2>&1 | grep -i "ModSecurity"
```

For a more detailed view, inspect the last 100 lines of logs:

```bash
docker logs --tail 100 waf
```

Expected output should contain entries indicating that ModSecurity processed the request, detected the SQL injection pattern, and applied the configured action (blocking).

#### Step 12 — Test with an alternative SQL injection pattern

Try another harmless test pattern to demonstrate consistent WAF behavior:

```bash
curl -i "http://localhost:8082/?search=%27%20OR%20%271%27%3D%271"
```

This decodes to:
```
GET /?search=' OR '1'='1
```

Expected output should also return HTTP 403 Forbidden, showing that the WAF blocks different variations of SQL injection patterns.

#### Step 13 — Verify WAF configuration details

Confirm that ModSecurity and the configured rule set are active:

```bash
docker exec waf nginx -T 2>&1 | grep -i modsecurity
```

Inspect the environment variables that were passed to the WAF container:

```bash
docker inspect waf --format '{{json .Config.Env}}' | grep -E "BLOCKING_PARANOIA|ANOMALY_INBOUND|BACKEND"
```

Expected output should show:
```
BLOCKING_PARANOIA=1
ANOMALY_INBOUND=5
BACKEND=http://backend:80
SERVERNAME=localhost
```

These settings control how strictly the WAF inspects requests and how the backend is reached.

#### Step 14 — Summary of results

The WAF deployment successfully demonstrates application-layer security:

| Test | Request | Result | HTTP Status |
|------|---------|--------|-------------|
| Legitimate traffic | `GET /` | Forwarded to backend | 200 OK |
| SQL injection variant 1 | `GET /?id=1 OR 1=1` | Blocked by ModSecurity | 403 Forbidden |
| SQL injection variant 2 | `GET /?search=' OR '1'='1` | Blocked by ModSecurity | 403 Forbidden |

#### Step 15 — Cleanup (optional)

When finished with the expansion, remove the containers and network:

```bash
docker compose down
```

To also remove the downloaded images:

```bash
docker image rm owasp/modsecurity-crs:nginx nginx:alpine
```

#### Result and Observation

The Web Application Firewall deployment using ModSecurity and the OWASP Core Rule Set successfully demonstrated application-layer security inspection. 

**Key achievements:**
1. **Legitimate requests forwarded:** Normal HTTP requests passed through the WAF without interference, with the backend returning HTTP 200.
2. **Malicious requests blocked:** Requests containing SQL injection patterns were detected by ModSecurity and blocked with HTTP 403 Forbidden.
3. **Rule detection logging:** The WAF logs confirmed that ModSecurity applied OWASP CRS rules to each request and enforced blocking actions.
4. **Network integration:** The WAF and backend communicated reliably over the Docker network, with proper DNS resolution and containerized networking.

**Security implications:**
This expands the defense-in-depth strategy beyond network and host hardening. While Task 4 (network segmentation) prevented lateral movement and Task 5 (firewall rules) restricted network exposure, the WAF adds **application-layer inspection**. Even if an attacker bypasses network controls or crafts a request that passes host-level firewalls, the WAF identifies and blocks common web application attacks before they reach the backend. The ModSecurity engine and OWASP CRS rules continuously evolve to detect new attack patterns, making the WAF a dynamic defense mechanism.

#### Evidence

![Expansion 1.1: Containers running and services initialized](img/ex1.3.png)
**Evidence placeholder:** Paste a screenshot showing `docker ps` output with both `waf` and `waf-backend` containers running.

---

![Expansion 1.2: Legitimate request forwarded (HTTP 200)](img/ex1.6.png)
**Evidence placeholder:** Paste a screenshot showing the output of `curl -i http://localhost:8082/` with `HTTP/1.1 200 OK` status.

---

![Expansion 1.3: Malicious SQL injection request blocked (HTTP 403)](img/ex1.6.png)
**Evidence placeholder:** Paste a screenshot showing the output of `curl -i "http://localhost:8082/?id=1%20OR%201%3D1"` with `HTTP/1.1 403 Forbidden` status.

--- 

![Expansion 1.4: WAF logs showing ModSecurity detection](img/ex1.8.png)
**Evidence placeholder:** Paste a screenshot showing the output of `docker logs waf` with entries indicating ModSecurity rule evaluation and blocking actions.

---

![Expansion 1.5: WAF configuration verification](img/ex1.10.png)

**Evidence placeholder:** Paste a screenshot showing the output of `docker inspect waf --format '{{json .Config.Env}}'` displaying the ModSecurity configuration parameters.

---

## Expansion 2 - Fail2ban for Repeated Failed Logins

**Objective:** Deploy Fail2ban to automatically block client IP addresses after repeated HTTP Basic Authentication failures, adding a brute-force defense layer.

#### Architecture Overview

The Fail2ban deployment monitors Nginx authentication logs and automatically blocks IP addresses that exceed a failure threshold:

```
┌─────────────────────┐
│    Kali Linux       │
│   (curl client)     │
└──────────┬──────────┘
           │
           ▼
    Nginx :8080
    (HTTP Basic Auth)
           │
      ┌────┴────────────┐
      │                 │
   Success          Failure
   (200 OK)         (401 Unauthorized)
      │                 │
      │                 ▼
      │          Nginx access.log
      │                 │
      │                 ▼
      │            Fail2ban
      │          (monitor logs)
      │                 │
      │            5 failures
      │            in 10 min
      │                 │
      │                 ▼
      │            BAN IP (iptables)
      │                 │
      └────────┬────────┘
               │
         All traffic
         from banned IP
         BLOCKED
```

Fail2ban runs in a companion Docker container, monitors the shared Nginx log directory, and uses iptables rules to block IPs that fail authentication more than 5 times in 10 minutes.

#### Step 1 — Create the Fail2ban lab directory structure

Set up the directory hierarchy for Nginx, Fail2ban filters, and jails:

```bash
mkdir -p ~/fail2ban-lab/nginx
mkdir -p ~/fail2ban-lab/fail2ban/filter.d
mkdir -p ~/fail2ban-lab/fail2ban/jail.d
cd ~/fail2ban-lab
```

Verify your location:

```bash
pwd
```

Expected output:
```
/home/paan/fail2ban-lab
```

Note: Your username and path may differ based on your system configuration.

#### Step 2 — Create the Nginx authentication configuration

Create the Nginx configuration file that requires HTTP Basic Authentication:

```bash
nano nginx/default.conf
```

Paste the following configuration:

```nginx
server {
    listen 80;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    location / {
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;

        return 200 "Authenticated OK\n";
    }
}
```

Save the file with `Ctrl + O`, press `Enter`, and exit with `Ctrl + X`.

This configuration:
- Listens on port 80
- Logs all requests to `/var/log/nginx/access.log`
- Requires HTTP Basic Authentication for all requests
- Returns HTTP 200 with "Authenticated OK" for successful authentication
- Automatically returns HTTP 401 Unauthorized for failed authentication

#### Step 3 — Create the HTTP Basic Authentication password file

Generate a bcrypt-hashed password for the test user:

```bash
docker run --rm httpd:alpine htpasswd -nbB student 'P@ssw0rd!' > nginx/htpasswd.txt
```

Verify the password file was created:

```bash
cat nginx/htpasswd.txt
```

Expected output (the hash will be unique):
```
student:$2y$05$...long hash...
```

The password is stored as a bcrypt hash, not plaintext.

#### Step 4 — Create the Fail2ban filter for Nginx authentication failures

Create a custom Fail2ban filter that recognizes HTTP 401 responses:

```bash
nano fail2ban/filter.d/nginx-auth.conf
```

Paste the following filter definition:

```ini
[Definition]

failregex = ^<HOST> .* "(GET|POST|HEAD) .* HTTP/.*" 401 .*

ignoreregex =
```

Save the file with `Ctrl + O`, press `Enter`, and exit with `Ctrl + X`.

**How the filter works:**
- `^<HOST>` — Captures the client IP address at the start of the log line
- `(GET|POST|HEAD)` — Matches HTTP methods
- `401` — Specifically matches HTTP 401 Unauthorized responses
- When a line matches this pattern, Fail2ban records it as a failure for that IP

Example matched log line:
```
192.168.1.10 - - [24/Aug/2026:10:30:45 +0800] "GET / HTTP/1.1" 401 179
```

Fail2ban extracts the IP `192.168.1.10` and increments the failure count.

#### Step 5 — Create the Fail2ban jail configuration

Create the jail that defines the enforcement policy:

```bash
nano fail2ban/jail.d/nginx-auth.local
```

Paste the following jail configuration:

```ini
[nginx-auth]

enabled = true
filter = nginx-auth

logpath = /var/log/nginx/access.log

maxretry = 5
findtime = 600
bantime = 3600

backend = auto
```

Save the file with `Ctrl + O`, press `Enter`, and exit with `Ctrl + X`.

**Configuration parameters:**

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `enabled` | true | Jail is active |
| `filter` | nginx-auth | Use the filter created in Step 4 |
| `logpath` | /var/log/nginx/access.log | Monitor this log file |
| `maxretry` | 5 | Ban after 5 failures |
| `findtime` | 600 | Count failures within a 10-minute window |
| `bantime` | 3600 | Ban lasts 1 hour (3600 seconds) |
| `backend` | auto | Auto-detect log backend |

**Security logic:**
If an IP generates 5 or more 401 responses within 10 minutes, it is banned for 1 hour. During the ban, all traffic from that IP is rejected.

#### Step 6 — Create the Docker Compose file

Create the orchestration file to run Nginx and Fail2ban together:

```bash
nano docker-compose.yml
```

Paste the following configuration:

```yaml
services:

  nginx:
    image: nginx:alpine
    container_name: fail2ban-nginx
    ports:
      - "8080:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./nginx/htpasswd.txt:/etc/nginx/.htpasswd:ro
      - ./nginx/log:/var/log/nginx
    restart: unless-stopped

  fail2ban:
    image: crazymax/fail2ban:latest
    container_name: fail2ban
    cap_add:
      - NET_ADMIN
      - NET_RAW
    volumes:
      - ./nginx/log:/var/log/nginx:ro
      - ./fail2ban/filter.d:/etc/fail2ban/filter.d:ro
      - ./fail2ban/jail.d:/etc/fail2ban/jail.d:ro
      - ./fail2ban/data:/data
    depends_on:
      - nginx
    restart: unless-stopped
```

Save the file with `Ctrl + O`, press `Enter`, and exit with `Ctrl + X`.

**Configuration details:**

- **nginx service:**
  - Runs on port 8080 (mapped to container port 80)
  - Mounts the authentication config, password file, and log directory
  - Shares the log directory with Fail2ban

- **fail2ban service:**
  - Uses the `crazymax/fail2ban:latest` image
  - Requires `NET_ADMIN` and `NET_RAW` capabilities to manage iptables rules
  - Mounts the Nginx log as read-only for monitoring
  - Mounts the filter and jail configurations
  - Depends on Nginx starting first

#### Step 7 — Create the log and data directories

Create the directories where logs and Fail2ban state will be stored:

```bash
mkdir -p nginx/log
mkdir -p fail2ban/data
```

Verify the directory structure:

```bash
ls -la
```

Expected output:
```
drwxr-xr-x docker-compose.yml
drwxr-xr-x nginx/
drwxr-xr-x fail2ban/
```

#### Step 8 — Start the containers

Launch both Nginx and Fail2ban in the background:

```bash
docker compose up -d
```

Expected output:
```
[+] Running 2/2
 ✔ Container fail2ban-nginx  Started
 ✔ Container fail2ban        Started
```

Verify the containers are running:

```bash
docker ps
```

Expected output should show:
- Container `fail2ban-nginx` listening on `0.0.0.0:8080->80/tcp`
- Container `fail2ban` running

#### Step 9 — Test Nginx authentication

Test unauthenticated access (should fail):

```bash
curl -i http://localhost:8080/
```

Expected output:
```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="Restricted"
Content-Length: 179
```

Now test with correct credentials:

```bash
curl -i -u student:'P@ssw0rd!' http://localhost:8080/
```

Expected output:
```
HTTP/1.1 200 OK
Content-Length: 16

Authenticated OK
```

This confirms:
- Nginx is running
- Authentication is enforced
- Valid credentials grant access
- Invalid/missing credentials return 401

#### Step 10 — Examine the Nginx access log

Check what entries Nginx has logged:

```bash
cat nginx/log/access.log
```

Expected output similar to:
```
172.x.x.x - - [24/Aug/2026:10:30:45 +0800] "GET / HTTP/1.1" 401 179
172.x.x.x - student [24/Aug/2026:10:30:46 +0800] "GET / HTTP/1.1" 200 16
```

**Key observations:**
- Unauthenticated requests show status `401`
- Authenticated requests show username (`student`) and status `200`
- Fail2ban monitors this log to identify patterns

#### Step 11 — Check Fail2ban status

Verify that Fail2ban is running and has recognized the jail:

```bash
docker logs fail2ban
```

Look for startup messages.

Check the list of active jails:

```bash
docker exec fail2ban fail2ban-client status
```

Expected output:
```
Status
|- Number of jail:  1
`- Jail list: nginx-auth
```

Check the specific jail status:

```bash
docker exec fail2ban fail2ban-client status nginx-auth
```

Expected output (initially all zeros):
```
Status for the jail: nginx-auth
|- Filter
|  |- Currently failed: 0
|  |- Total failed: 0
|  |- File list: /var/log/nginx/access.log
|
`- Actions
   |- Currently banned: 0
   |- Total banned: 0
```

This confirms Fail2ban is monitoring the Nginx log.

#### Step 12 — Generate repeated failed authentication attempts

Simulate a brute-force attack by sending 5 requests with an incorrect password:

```bash
for i in {1..5}; do
    curl -s -o /dev/null -w "Attempt $i: HTTP %{http_code}\n" \
    -u student:'WrongPassword123!' \
    http://localhost:8080/
done
```

Expected output:
```
Attempt 1: HTTP 401
Attempt 2: HTTP 401
Attempt 3: HTTP 401
Attempt 4: HTTP 401
Attempt 5: HTTP 401
```

All five requests fail with HTTP 401.

#### Step 13 — Check Fail2ban jail status after failures

Wait a few seconds for Fail2ban to process the logs, then check the jail:

```bash
docker exec fail2ban fail2ban-client status nginx-auth
```

Expected output after the 5 failures:
```
Status for the jail: nginx-auth
|- Filter
|  |- Currently failed: 0
|  |- Total failed: 5
|  |- File list: /var/log/nginx/access.log
|
`- Actions
   |- Currently banned: 1
   |- Total banned: 1
```

**Key indicators:**
- `Total failed: 5` — Fail2ban counted all 5 failed attempts
- `Currently banned: 1` — One IP address is currently banned
- `Total banned: 1` — This IP has been banned

#### Step 14 — Identify the banned IP address

Retrieve the currently banned IP:

```bash
docker exec fail2ban fail2ban-client get nginx-auth banip
```

Expected output:
```
172.18.0.1
```

The exact IP will depend on Docker networking (this is the container's gateway or host IP as seen from Nginx).

#### Step 15 — Test a request from the banned IP

Attempt another request with the wrong password:

```bash
curl -v -u student:'WrongPassword123!' http://localhost:8080/
```

Expected behavior:
- The connection may hang or timeout
- Or you may see a connection reset
- Or Fail2ban may show the request was blocked

The exact result depends on how the Docker Fail2ban container enforces iptables rules. **The strongest evidence is the Fail2ban jail status itself**, which clearly shows the IP is banned.

#### Step 16 — Verify the Nginx log entries

Check the access log to confirm the failed attempts were logged:

```bash
cat nginx/log/access.log | tail -10
```

Expected output should include multiple 401 entries from your client IP:
```
172.x.x.x - - [...] "GET / HTTP/1.1" 401 179
172.x.x.x - - [...] "GET / HTTP/1.1" 401 179
172.x.x.x - - [...] "GET / HTTP/1.1" 401 179
172.x.x.x - - [...] "GET / HTTP/1.1" 401 179
172.x.x.x - - [...] "GET / HTTP/1.1" 401 179
```

#### Step 17 — Test the Fail2ban filter manually

Verify that the filter regex correctly identifies the failed-login log lines:

```bash
docker exec fail2ban fail2ban-regex \
    /var/log/nginx/access.log \
    /etc/fail2ban/filter.d/nginx-auth.conf
```

Expected output will show:
```
Lines: processed X, matched Y
```

Where Y represents the number of 401 entries matched by the filter. If Y ≥ 5, the filter is working correctly.

This test is valuable because it demonstrates the complete flow:
```
Nginx log → Fail2ban filter → Pattern match → Count failures
```

#### Step 18 — Check Fail2ban internal logs

View the Fail2ban daemon log to see ban events:

```bash
docker exec fail2ban sh -c 'tail -50 /var/log/fail2ban.log 2>/dev/null || true'
```

Look for lines containing `Ban`, such as:
```
2026-08-24 10:35:12,345 fail2ban.actions [12345]: NOTICE  [nginx-auth] Ban 172.x.x.x
```

This confirms the ban action was triggered.

#### Step 19 — Test with valid credentials (if ban allows)

If the ban has expired or if Fail2ban permits requests with valid credentials, test a successful login:

```bash
curl -i -u student:'P@ssw0rd!' http://localhost:8080/
```

If the IP is still banned, even valid credentials will be blocked (demonstrating the effectiveness of the ban).

#### Step 20 — Unban the test IP (to reset for re-testing)

To reset the lab and remove the ban, first identify the banned IP:

```bash
docker exec fail2ban fail2ban-client get nginx-auth banip
```

Suppose it returns `172.18.0.1`. Then unban it:

```bash
docker exec fail2ban fail2ban-client set nginx-auth unbanip 172.18.0.1
```

Verify the ban was removed:

```bash
docker exec fail2ban fail2ban-client status nginx-auth
```

Expected output should show:
```
Currently banned: 0
Total banned: 1
```

The `Total banned` counter remains at 1 (historical record), but `Currently banned` is now 0.

#### Step 21 — Cleanup (optional)

When finished with the expansion, stop and remove the containers:

```bash
docker compose down
```

To also remove the Docker image:

```bash
docker image rm crazymax/fail2ban:latest nginx:alpine
```

#### Result and Observation

The Fail2ban deployment successfully demonstrated automated brute-force defense for HTTP Basic Authentication:

**Key achievements:**
1. **Failure detection:** Nginx logged all 401 authentication failures to `access.log`.
2. **Pattern matching:** Fail2ban's custom filter correctly identified 401 responses from the client IP.
3. **Automatic banning:** After 5 failures within 10 minutes, the IP was automatically banned.
4. **Brute-force mitigation:** The banned IP could not establish new connections, preventing further login attempts.
5. **Persistent logging:** Fail2ban maintained a ban history and could display currently banned IPs.

**Security implications:**

This expansion adds **behavioral defense** beyond the static controls in the main lab:
- Task 1 enforced authentication (valid credentials required)
- Task 5 enforced network firewalls (explicit allow rules)
- **Expansion 2 adds dynamic blocking** (automatic response to attack patterns)

Fail2ban watches for repeated failures and responds faster than manual intervention. The default 5 failures in 10 minutes is configurable for different risk profiles. More aggressive settings (e.g., 3 failures in 5 minutes) are useful for high-value targets. Less aggressive settings avoid blocking legitimate users who mistype passwords.

The ban mechanism uses the underlying host's iptables, making the block **network-level** rather than application-level. A banned IP cannot even establish a TCP connection, so no application logic is involved in the rejection.

#### Evidence

![Expansion 2.1: Nginx authentication and failed attempts](img/ex2.24.png)
**Evidence placeholder:** Paste a screenshot showing the output of:
- `curl -i http://localhost:8080/` (401 Unauthorized)
- `curl -i -u student:'P@ssw0rd!' http://localhost:8080/` (200 OK)
- The 5 failed authentication attempts loop

---

![Expansion 2.2: Nginx access log showing 401 entries](img/ex2.20.png)
**Evidence placeholder:** Paste a screenshot showing `cat nginx/log/access.log` with multiple 401 entries from the client IP.

---

![Expansion 2.3: Fail2ban jail status showing ban](img/ex2.16.png)
**Evidence placeholder:** Paste a screenshot showing the output of:
- `docker exec fail2ban fail2ban-client status nginx-auth` with `Currently banned: 1` and `Total banned: 1`

---

![Expansion 2.4: Banned IP](img/ex2.17.png)
**Evidence placeholder:** Paste a screenshot showing:
- `docker exec fail2ban fail2ban-client get nginx-auth banip` (banned IP address)

---

![Expansion 2.5: Filter test and Fail2ban logs](img/ex2.21.png)
**Evidence placeholder:** Paste a screenshot showing the output of:
- `docker exec fail2ban fail2ban-regex /var/log/nginx/access.log /etc/fail2ban/filter.d/nginx-auth.conf` (showing matched lines)


---

## Expansion 3 - Service Mesh with Mutual TLS using Istio

**Objective:** Deploy Istio service mesh and enforce mutual TLS (mTLS) between application services, implementing zero-trust network security where all service-to-service traffic is encrypted and authenticated.

#### Architecture Overview

Istio provides automatic encryption and authentication for service-to-service communication through sidecar proxies (Envoy). Application workloads communicate through their Envoy sidecars, which establish mTLS connections secured by workload certificates issued by the mesh CA (Certificate Authority):

```
                    Kali Linux
                        │
                        │ kubectl
                        │
                        ▼
              ┌──────────────────────┐
              │  Kubernetes (kind)   │
              │                      │
              │  ┌────────────────┐  │
              │  │  Istiod (CA)   │  │
              │  └────────────────┘  │
              │         │            │
              │   distributes certs  │
              │         │            │
              │    ┌────┴─────┐      │
              │    │          │      │
              │  ┌─▼──────┐ ┌─▼──────┐
              │  │ Client │ │Backend │
              │  │  App   │ │  App   │
              │  │        │ │        │
              │  │ Envoy  │─│ Envoy  │ mTLS
              │  └────────┘ └────────┘
              │  (2/2 Ready) (2/2 Ready)
              │                      │
              │  ┌────────────────┐  │
              │  │ Outside Client │  │
              │  │    (no mesh)   │  │
              │  │   1/1 Ready    │  │
              │  └────────────────┘  │
              │                      │
              └──────────────────────┘
```

When a client pod sends a request to a backend pod, the client's Envoy sidecar:
1. Intercepts the application's plaintext HTTP request
2. Establishes an mTLS connection to the backend's Envoy
3. Forwards the plaintext request through the encrypted tunnel
4. Backend's Envoy decrypts and delivers to the backend application

This transparent encryption means applications use HTTP while the infrastructure layer provides mTLS.

#### Phase 1: Cluster Preparation

##### Step 1 — Verify existing environment

Ensure Docker, kind, and kubectl are available:

```bash
docker --version
kind --version
kubectl version --client
```

Check the current Kubernetes context:

```bash
kubectl config current-context
```

You may have your existing `kind-ccse-lab4` cluster from the main lab. Do not delete it.

##### Step 2 — Create a dedicated Istio cluster

Create a separate kind cluster to avoid disrupting your main lab work:

```bash
kind create cluster --name istio-lab
```

Verify it was created:

```bash
kind get clusters
```

Expected output:
```
ccse-lab4
istio-lab
```

Switch kubectl to the new cluster:

```bash
kubectl config use-context kind-istio-lab
```

Verify:

```bash
kubectl config current-context
kubectl get nodes
```

Expected output:
```
kind-istio-lab

NAME                       STATUS   ROLES           AGE
istio-lab-control-plane    Ready    control-plane   xx...
```

#### Phase 2: Istio Installation

##### Step 3 — Install Istio CLI

Download and install the Istio command-line tool:

```bash
curl -sL https://istio.io/downloadIstioctl | sh -
export PATH=$HOME/.istioctl/bin:$PATH
```

Verify installation:

```bash
istioctl version
```

Expected output:
```
client version: 1.30.x (or newer)
```

##### Step 4 — Deploy Istio on the cluster

Install Istio using the demo profile (appropriate for testing and learning):

```bash
istioctl install --set profile=demo -y
```

Expected output:
```
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Installation complete
```

Verify the installation:

```bash
kubectl get pods -n istio-system
```

Expected output should show:
```
NAME                                      READY   STATUS
istiod-xxxxxxxxxx-xxxxx                   1/1     Running
istio-ingressgateway-xxxxxxxxxx-xxxxx     1/1     Running
istio-egressgateway-xxxxxxxxxx-xxxxx      1/1     Running
```

Verify with istioctl:

```bash
istioctl verify-install
```

Expected output:
```
✔ Istio is installed and verified successfully
```

#### Phase 3: Application Deployment with Sidecar Injection

##### Step 5 — Create the application namespace

```bash
kubectl create namespace app
```

##### Step 6 — Enable automatic Envoy sidecar injection

Label the namespace to enable automatic injection of Envoy sidecar proxies:

```bash
kubectl label namespace app istio-injection=enabled
```

Verify:

```bash
kubectl get namespace app --show-labels
```

Expected output:
```
NAME   STATUS   AGE   LABELS
app    Active   xx    istio-injection=enabled
```

**Important:** Pods must be created AFTER applying this label to receive sidecars. Existing pods are not automatically modified.

##### Step 7 — Deploy the backend service

Create a backend application with an Envoy sidecar:

```bash
kubectl apply -n app -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: nginxdemos/hello:plain-text
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
EOF
```

Wait for the backend to be ready:

```bash
kubectl get deployment -n app
kubectl get pods -n app
```

Expected output:
```
NAME      READY   UP-TO-DATE   AVAILABLE
backend   1/1     1            1

NAME                       READY   STATUS
backend-xxxxxxxxxx-xxxxx   2/2     Running
```

**Important:** The `2/2` ready status shows that each pod has 2 containers:
1. The backend application container
2. The istio-proxy (Envoy sidecar) container

##### Step 8 — Deploy a client workload inside the mesh

Create a client pod to test communication:

```bash
kubectl run client \
  -n app \
  --image=curlimages/curl \
  --restart=Never \
  --command -- sleep 3600
```

Wait for it to be ready:

```bash
kubectl wait --for=condition=Ready pod/client -n app --timeout=120s
```

Verify:

```bash
kubectl get pods -n app
```

Expected output:
```
NAME                       READY   STATUS
backend-xxxxxxxxxx-xxxxx   2/2     Running
client                     2/2     Running
```

Both pods have `2/2` containers (application + Envoy sidecar) because both are in the `app` namespace with sidecar injection enabled.

Verify sidecars are properly injected:

```bash
kubectl get pod -n app -o jsonpath='{.items[*].spec.containers[*].name}'; echo
```

Expected output should show both `backend` and `istio-proxy` in each pod.

#### Phase 4: mTLS Enforcement and Testing

##### Step 9 — Apply STRICT mTLS policy

Enforce the security requirement that all service-to-service traffic must use mTLS:

```bash
kubectl apply -n app -f - <<'EOF'
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
EOF
```

Verify the policy:

```bash
kubectl get peerauthentication -n app
```

Expected output:
```
NAME      MODE
default   STRICT
```

Display the full policy:

```bash
kubectl get peerauthentication default -n app -o yaml
```

This shows the mTLS requirement is active.

##### Step 10 — Test in-mesh communication (should succeed)

Send a request from the client pod (inside the mesh) to the backend (inside the mesh):

```bash
kubectl exec -n app client -c client -- curl -s http://backend
```

Expected output (a successful response from the Nginx backend):
```
Server address: 10.x.x.x:80
Server name: backend-xxxxxxxxxx-xxxxx
Date: ...
URI: /
Request ID: ...
```

**Why this succeeds:** Both client and backend have Envoy sidecars. The client's Envoy establishes an mTLS connection to the backend's Envoy, and the traffic is encrypted and authenticated.

##### Step 11 — Verify Istio certificate distribution

Confirm that the workloads received certificates from the Istio CA:

```bash
istioctl proxy-config secret client -n app
```

Expected output:
```
RESOURCE NAME
default
ROOTCA
```

Check the backend as well:

```bash
BACKEND_POD=$(kubectl get pods -n app -l app=backend -o jsonpath='{.items[0].metadata.name}')
istioctl proxy-config secret "$BACKEND_POD" -n app
```

This confirms that Istio distributed workload certificates to both proxies.

##### Step 12 — Verify proxy status

Check that both proxies are synced with the control plane:

```bash
istioctl proxy-status
```

Expected output should show:
```
NAME                                                   CLUSTER       CDS   LDS   EDS   RDS
backend-xxxxxxxxxx-xxxxx.app                           Kubernetes    SYNCED SYNCED SYNCED SYNCED
client.app                                             Kubernetes    SYNCED SYNCED SYNCED SYNCED
```

All proxies showing `SYNCED` indicates the mesh is properly configured.

##### Step 13 — Verify Envoy configuration

Check the client's Envoy cluster configuration to confirm it discovered the backend:

```bash
istioctl proxy-config clusters client -n app | grep backend
```

Expected output:
```
backend.app.svc.cluster.local    80    -    outbound    EDS
```

This proves the client's Envoy proxy has discovered and is ready to communicate with the backend service.

#### Phase 5: Zero-Trust Verification (Critical Test)

This phase demonstrates that STRICT mTLS actually enforces the security policy by rejecting plaintext traffic from outside the mesh.

##### Step 14 — Create a namespace outside the mesh

Create a namespace that does NOT have sidecar injection enabled:

```bash
kubectl create namespace outside
```

##### Step 15 — Deploy a client outside the mesh

Create a client pod in the `outside` namespace (no Envoy sidecar):

```bash
kubectl run outside-client \
  -n outside \
  --image=curlimages/curl \
  --restart=Never \
  --command -- sleep 3600
```

Wait for it:

```bash
kubectl wait --for=condition=Ready pod/outside-client -n outside --timeout=120s
```

Verify:

```bash
kubectl get pods -n outside
```

Expected output:
```
NAME              READY   STATUS
outside-client    1/1     Running
```

**Critical observation:** `1/1` (only application container, no Envoy sidecar) because this namespace does not have `istio-injection=enabled`.

##### Step 16 — Test plaintext traffic from outside the mesh (should fail)

Attempt to connect from the non-mesh client to the backend service:

```bash
kubectl exec -n outside outside-client -- \
  curl -v --max-time 5 http://backend.app.svc.cluster.local
```

Expected behavior:
- Connection times out or is refused
- No successful response from the backend
- The request may see an error or hang

**Why this fails:** The backend's Envoy sidecar has a STRICT mTLS policy. It only accepts mTLS connections from clients that have a valid Istio-issued certificate. The outside-client has no Envoy sidecar and cannot present a certificate, so its plaintext HTTP request is rejected at the network level.

**Important evidence:** This test proves that the STRICT mTLS policy is actually enforcing security, not just configured. A non-mesh client cannot communicate even though it's in the same Kubernetes cluster.

##### Step 17 — Compare the two scenarios

You now have two critical test results:

| Test Scenario | Client | Backend | Result | Reason |
|---------------|--------|---------|--------|--------|
| In-mesh | Envoy sidecar ✓ | Envoy sidecar ✓ | SUCCESS | Both have certificates; mTLS tunnel established |
| Out-of-mesh | No sidecar ✗ | Envoy sidecar ✓ | FAILED | Client cannot present certificate; STRICT policy rejects |

This comparison is the strongest evidence that zero-trust is working.

#### Phase 6: Final Verification and Cleanup

##### Step 18 — Comprehensive mesh verification

Run a complete verification command:

```bash
echo "=== ISTIO SYSTEM ===" && \
kubectl get pods -n istio-system && \
echo && \
echo "=== APPLICATION NAMESPACE ===" && \
kubectl get pods -n app && \
echo && \
echo "=== MTLS POLICY ===" && \
kubectl get peerauthentication -n app && \
echo && \
echo "=== PROXY STATUS ===" && \
istioctl proxy-status
```

This provides a comprehensive overview of the entire mesh setup.

##### Step 19 — Analyze the mesh

Run Istio's validation tool:

```bash
istioctl analyze -n app
```

Expected output:
```
✔ No validation issues found when analyzing namespace: app.
```

##### Step 20 — Cleanup (optional)

After capturing evidence, remove the lab:

```bash
# Remove application workloads
kubectl delete namespace app
kubectl delete namespace outside

# Uninstall Istio
istioctl uninstall --purge -y

# Delete the cluster
kind delete cluster --name istio-lab

# Verify your original cluster is untouched
kind get clusters
```

Your original `ccse-lab4` remains intact.

#### Result and Observation

The Istio service mesh deployment successfully demonstrated zero-trust service-to-service security:

**Key achievements:**
1. **Service mesh deployment:** Istio control plane (Istiod) installed and managing the cluster
2. **Automatic sidecar injection:** Application pods received Envoy proxies without manual intervention
3. **Workload identity:** Each pod received a unique certificate from the Istio CA
4. **Encrypted communication:** In-mesh services communicated through mTLS without application code changes
5. **mTLS enforcement:** STRICT policy rejected plaintext traffic from outside the mesh
6. **Zero-trust demonstrated:** Communication required both valid identity (certificate) and encryption (mTLS)

**Security implications:**

This expansion implements the **highest layer of the defense-in-depth strategy**:
- Task 1 (AuthN) verified **who** can access a service
- Task 4 (network segmentation) prevented **which** services could reach each other
- **Expansion 3 adds mutual authentication and encryption** between services that can reach each other

The zero-trust model states: *never trust, always verify*. Istio implements this by:
- Issuing cryptographic identity (certificates) to every workload
- Requiring mTLS for all service-to-service traffic
- Transparently encrypting and authenticating without application changes
- Automatically rejecting traffic that doesn't meet the policy

A compromised service inside the mesh still cannot impersonate another service—it would need the other service's private key. Even if an attacker gains code execution in a pod, they cannot decrypt traffic to other services because that traffic is encrypted by the Envoy sidecar before leaving the pod.

#### Evidence

![Expansion 3.1: Istio installation and components](img/ex3.5.png)
**Evidence placeholder:** Paste a screenshot showing:
- `kubectl get pods -n istio-system` (istiod, ingress gateway, egress gateway)
- `istioctl analyze -A` (showing successful verification)

---

![Expansion 3.2: Sidecar injection and pod status](img/ex3.9.png)
**Evidence placeholder:** Paste a screenshot showing:
- `kubectl get pods -n app` (both backend and client with 2/2 status)
- `istioctl proxy-status` (both workloads listed as SYNCED)

---

![Expansion 3.3: STRICT mTLS policy configuration](img/ex3.12.png)
**Evidence placeholder:** Paste a screenshot showing:
- `kubectl get peerauthentication -n app` (policy name and mode)
- `kubectl get peerauthentication default -n app -o yaml` (full policy definition)

---

![Expansion 3.4: Successful in-mesh communication](img/ex3.13.png)
**Evidence placeholder:** Paste a screenshot showing:
- `kubectl exec -n app client -c client -- curl -s http://backend` (successful response)
- Backend service response with Server address and metadata

---

![Expansion 3.5: Out-of-mesh connection rejection (zero-trust enforcement)](img/ex3.25.png)
**Evidence placeholder:** Paste a screenshot showing:
- `kubectl exec -n app external-client -- \
  curl -i --max-time 5 http://backend.app.svc.cluster.local/` (connection timeout or refusal)
- This proves STRICT mTLS is rejecting plaintext traffic

---

![Expansion 3.6: Certificate and proxy verification](img/ex3.26.png)
**Evidence placeholder:** Paste a screenshot showing:
- `istioctl proxy-config secret client -n app` (showing workload certificates)
- `istioctl proxy-config clusters client -n app | grep backend` (backend discovery)

---

## Expansion 4 - Distroless Hardened Image Using Minimal Runtime

**Objective:** Build a reproducible, minimal Docker image using a multi-stage Dockerfile and the `scratch` base image, demonstrating how to reduce container attack surface by removing unnecessary OS components, package managers, and interactive shells.

#### Architecture Overview

The expansion demonstrates a minimal container approach using multi-stage builds and the `scratch` base image:

```
Build Stage              Runtime Stage
┌──────────────────┐     ┌────────────────────┐
│   BusyBox Image  │     │   scratch (empty)  │
│                  │     │                    │
│  - Full OS       │     │  - Nothing by      │
│  - Package mgr   │     │    default         │
│  - Utilities     │     │                    │
└────────┬─────────┘     └────────┬───────────┘
         │                        │
    Extract                   Copy only
    /bin/busybox             BusyBox + app
         │                        │
         └───────────┬────────────┘
                     │
                     ▼
            ┌────────────────────┐
            │  Final Image       │
            │                    │
            │  /bin/busybox      │
            │  /www/index.html   │
            │                    │
            │  UID 65534:65534   │
            │  No shell          │
            │  No package mgr    │
            │  ~1-5 MB           │
            └────────────────────┘
```

The multi-stage build extracts only the required BusyBox binary and application files, resulting in an extremely minimal runtime image.

#### Phase 1: Dockerfile Creation

##### Step 1 — Create the working directory

```bash
mkdir -p ~/distroless-lab
cd ~/distroless-lab
pwd
```

Expected output:
```
/home/paan/distroless-lab
```

##### Step 2 — Create the Dockerfile

Create a multi-stage Dockerfile that uses BusyBox as the build stage and `scratch` as the final runtime:

```bash
nano Dockerfile
```

Paste the following:

```dockerfile
# Build stage: obtain BusyBox
FROM busybox:1.36-musl AS builder

# Runtime stage: completely empty base
FROM scratch

# Copy only the required BusyBox binary
COPY --from=builder /bin/busybox /bin/busybox

# Copy the application files
COPY www /www

# Run as nobody (UID/GID 65534)
USER 65534:65534

# Application port
EXPOSE 8080

# Start BusyBox HTTP server
ENTRYPOINT ["/bin/busybox", "httpd", "-f", "-v", "-p", "8080", "-h", "/www"]
```

Save with `Ctrl + O`, press `Enter`, and exit with `Ctrl + X`.

**Key design choices:**
- `FROM scratch` — The final image starts with absolutely nothing (no OS, no shell, no package manager)
- `COPY --from=builder` — Only the BusyBox binary is copied from the build stage
- `USER 65534:65534` — Configured to run as non-root UID 65534 (conventionally "nobody")
- `ENTRYPOINT` — Fixed application command (no interactive shell available)

This approach results in a filesystem containing only:
```
/bin/busybox
/www/index.html
```

##### Step 3 — Create the application content

Create a simple HTML application to serve:

```bash
mkdir -p www
```

Create the index.html file:

```bash
cat > www/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>CCSE Distroless Lab</title>
</head>
<body>
    <h1>Distroless Hardened Container</h1>
    <p>Expansion 4 - Minimal Container Security Test</p>
</body>
</html>
EOF
```

Verify:

```bash
cat www/index.html
```

#### Phase 2: Image Building

##### Step 4 — Build the minimal image

```bash
docker build -t ccse-hardened:distroless .
```

Expected output:
```
Successfully tagged ccse-hardened:distroless
```

Verify the image was created:

```bash
docker images ccse-hardened
```

Expected output:
```
REPOSITORY       TAG           IMAGE ID       CREATED       SIZE
ccse-hardened    distroless    xxxxxxxxxxxx   x second ago   xxx KB
```

**Note:** The image size should be extremely small compared to a full Linux base image (often < 5 MB).

##### Step 5 — Inspect the image metadata

Verify the image is configured for non-root execution:

```bash
docker image inspect ccse-hardened:distroless \
    --format 'User={{.Config.User}}'
```

Expected output:
```
User=65534:65534
```

Verify the entrypoint is set to the BusyBox HTTP server:

```bash
docker image inspect ccse-hardened:distroless \
    --format 'Entrypoint={{json .Config.Entrypoint}}'
```

Expected output:
```
Entrypoint=["/bin/busybox","httpd","-f","-v","-p","8080","-h","/www"]
```

#### Phase 3: Runtime Deployment and Testing

##### Step 6 — Start the hardened container

Clean up any previous instances:

```bash
docker rm -f hardened-distroless 2>/dev/null
```

Start the container with the security controls from Task 6:

```bash
docker run -d \
    --name hardened-distroless \
    --read-only \
    --cap-drop=ALL \
    --security-opt no-new-privileges \
    --tmpfs /tmp \
    -p 8081:8080 \
    ccse-hardened:distroless
```

Verify it's running:

```bash
docker ps
```

Expected output:
```
CONTAINER ID   IMAGE                          COMMAND                  PORTS
xxxxxxxxxxxx   ccse-hardened:distroless      "/bin/busybox httpd..."  0.0.0.0:8081->8080/tcp
```

##### Step 7 — Test the application

Send an HTTP request to the running container:

```bash
curl -i http://localhost:8081/
```

Expected output:
```
HTTP/1.0 200 OK
Content-type: text/html

<!DOCTYPE html>
<html>
<head>
    <title>CCSE Distroless Lab</title>
</head>
<body>
    <h1>Distroless Hardened Container</h1>
    <p>Expansion 4 - Minimal Container Security Test</p>
</body>
</html>
```

This proves the minimal container successfully serves the application despite having no shell or package manager.

##### Step 8 — Verify non-root execution

```bash
docker inspect hardened-distroless \
    --format 'User={{.Config.User}}'
```

Expected output:
```
User=65534:65534
```

The container runs as UID 65534 (nobody), not root (UID 0).

##### Step 9 — Verify read-only filesystem

```bash
docker inspect hardened-distroless \
    --format 'ReadOnly={{.HostConfig.ReadonlyRootfs}}'
```

Expected output:
```
ReadOnly=true
```

##### Step 10 — Verify capability dropping

```bash
docker inspect hardened-distroless \
    --format 'CapDrop={{json .HostConfig.CapDrop}}'
```

Expected output:
```
CapDrop=["ALL"]
```

All Linux capabilities have been removed.

##### Step 11 — Verify no-new-privileges

```bash
docker inspect hardened-distroless \
    --format 'SecurityOpt={{json .HostConfig.SecurityOpt}}'
```

Expected output:
```
SecurityOpt=["no-new-privileges"]
```

Processes cannot gain additional privileges, even through setuid binaries.

##### Step 12 — Verify temporary storage

```bash
docker inspect hardened-distroless \
    --format 'Tmpfs={{json .HostConfig.Tmpfs}}'
```

Expected output:
```
Tmpfs={"/tmp":""}
```

The container has `/tmp` as temporary writable storage while the root filesystem remains read-only.

##### Step 13 — Comprehensive security verification

Display all security settings in one command:

```bash
docker inspect hardened-distroless \
    --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}} CapDrop={{json .HostConfig.CapDrop}} SecurityOpt={{json .HostConfig.SecurityOpt}} Tmpfs={{json .HostConfig.Tmpfs}}'
```

Expected output:
```
User=65534:65534 ReadOnly=true CapDrop=["ALL"] SecurityOpt=["no-new-privileges"] Tmpfs={"/tmp":""}
```

#### Phase 4: Minimal Attack Surface Verification

##### Step 14 — Verify absence of shell

Attempt to execute a shell inside the container:

```bash
docker exec hardened-distroless /bin/sh
```

Expected result:
```
Error: No such file or directory
```

or similar error. This is expected and demonstrates a security advantage.

**Why this is important:** The container cannot be interactively accessed because the `scratch` runtime image contains no shell. An attacker gaining code execution cannot easily explore the filesystem or escalate privileges. Applications must be debugged through logs and monitoring, not interactive shells.

##### Step 15 — Verify absence of package managers

Verify that common package managers are not available:

```bash
docker exec hardened-distroless /bin/apk 2>&1
docker exec hardened-distroless /bin/apt 2>&1
```

Expected result:
```
Error: No such file or directory
```

Package managers increase attack surface because they provide tools for installing malicious software. Their absence is a hardening benefit.

##### Step 16 — Check container logs

View any messages from the HTTP server:

```bash
docker logs hardened-distroless
```

This shows that the application is functioning and logging output normally despite the minimal runtime.

#### Phase 5: Vulnerability Assessment

##### Step 17 — Scan the distroless image

Use Trivy to scan the built image for known vulnerabilities:

```bash
trivy image --severity HIGH,CRITICAL ccse-hardened:distroless
```

Record the actual results. Example output:
```
2 vulnerabilities found (HIGH: 1, CRITICAL: 0)
```

##### Step 18 — Compare against nginx:alpine

Scan the reference image from Task 6 for comparison:

```bash
trivy image --severity HIGH,CRITICAL nginx:alpine
```

Record the actual results. Example output:
```
5 vulnerabilities found (HIGH: 3, CRITICAL: 0)
```

**Comparison table:**

| Image | HIGH | CRITICAL |
|-------|------|----------|
| nginx:alpine | Insert actual result | Insert actual result |
| ccse-hardened:distroless | Insert actual result | Insert actual result |

The distroless image typically has fewer or equal vulnerabilities because it includes far fewer packages.

#### Security Analysis

**Security measures applied:**

| Control | Implementation | Security Effect |
|---------|------------------|------------------|
| Minimal runtime | FROM scratch | Removes OS, shells, package managers |
| Multi-stage build | BusyBox → scratch | Prevents build tools in final image |
| Non-root execution | USER 65534:65534 | Prevents root-level compromise |
| Read-only filesystem | --read-only | Blocks persistence and binary modification |
| Capability dropping | --cap-drop=ALL | Removes privileged Linux operations |
| No new privileges | --security-opt no-new-privileges | Prevents privilege escalation |
| Temporary storage | --tmpfs /tmp | Allows app writes without root FS writable |
| No interactive shell | scratch runtime | Blocks interactive post-compromise access |
| Vulnerability scanning | Trivy | Identifies known HIGH/CRITICAL issues |

**Comparison with Task 6:**

- **Task 6** used `nginxinc/nginx-unprivileged` (a pre-built hardened image)
- **Expansion 4** builds a custom minimal image from scratch

This demonstrates how to:
- Apply the same hardening controls to a custom application
- Use multi-stage builds to minimize image size
- Reduce attack surface by removing unnecessary components
- Build reproducible, minimal, secure container images

#### Result and Observation

The distroless Dockerfile successfully created a minimal, hardened container image with the following characteristics:

**Achievement summary:**
1. **Minimal runtime:** Image contains only BusyBox binary and application files
2. **No shell:** Interactive compromise is prevented by absence of /bin/sh
3. **No package manager:** Prevents installation of malicious tools
4. **Non-root execution:** Application runs as UID 65534 (nobody), not root
5. **Read-only root filesystem:** Prevents unauthorized filesystem modifications
6. **Dropped capabilities:** All Linux capabilities removed
7. **No privilege escalation:** `no-new-privileges` prevents setuid exploitation
8. **Temporary storage:** `/tmp` is writable while root filesystem is read-only
9. **Small image size:** Minimal runtime reduces download time and storage requirements
10. **Reduced vulnerabilities:** Fewer packages typically means fewer known CVEs

**Security implications:**

This expansion demonstrates **supply-chain and runtime hardening**:
- The Dockerfile is reproducible and auditable
- Build outputs are predictable and minimal
- Runtime attack surface is vastly reduced compared to full Linux distributions
- Even if an attacker gains code execution, tools and privileges are extremely limited
- Absence of package managers prevents installing persistence mechanisms

**Real-world applications:**

Minimal container images like this are used in production for:
- Serverless functions (AWS Lambda uses similar minimal runtimes)
- Distroless images by Google (official minimal images for various runtimes)
- Security-hardened microservices
- Cost optimization (smaller images = faster deployment, less storage)

#### Evidence

![Expansion 4.1: Multi-stage Dockerfile with scratch runtime](img/ex4.5.png)
**Evidence placeholder:** Paste a screenshot showing:
- `cat Dockerfile` output
- Multi-stage build structure (FROM busybox AS builder, FROM scratch)
- COPY --from=builder and USER directives

---

![Expansion 4.2: Successful image build](img/ex4.6.png)
**Evidence placeholder:** Paste a screenshot showing:
- `docker build -t ccse-hardened:distroless .` (successful build)
- `docker images ccse-hardened` (image listed with size)

---

![Expansion 4.3: Application running (HTTP 200 OK)](img/ex4.9.png)
**Evidence placeholder:** Paste a screenshot showing:
- `curl -i http://localhost:8081/` (HTTP/1.0 200 OK response)
- HTML content displayed (Distroless Hardened Container heading)

---

![Expansion 4.4: Comprehensive security verification](img/ex4.10.png)
**Evidence placeholder:** Paste a screenshot showing all security settings:
- User=65534:65534
- ReadOnly=true
- CapDrop=["ALL"]
- SecurityOpt=["no-new-privileges"]
- Tmpfs={"/tmp":""}

---

![Expansion 4.5: Vulnerability scan comparison](img/ex4.14.png)
![Expansion 4.5: Vulnerability scan comparison](img/ex4.15.png)
**Evidence placeholder:** Paste a screenshot showing:
- `trivy image --severity HIGH,CRITICAL ccse-hardened:distroless` (result)
- `trivy image --severity HIGH,CRITICAL nginx:alpine` (result for comparison)
- Comparison table with actual HIGH and CRITICAL counts

## Deliverables and Assessment Evidence

| Task | Evidence to include |
|------|---------------------|
| Task 1 | HTTP 401 without credentials and HTTP 200 with valid credentials |
| Task 2 | Generated TOTP test and `MFA OK` |
| Task 3 | `yes`, `no`, `no` RBAC results |
| Task 4 | `web -> db BLOCKED` and `app -> db REACHABLE` |
| Task 5 | iptables INPUT policy DROP and explicit ACCEPT rules |
| Task 6 | Hardened container inspect output and Trivy summary |
| Verification | RoleBinding YAML and capability-drop output |
| Expansions | WAF, Fail2ban, Istio mTLS and distroless evidence, if executed |

## Security Best-Practices Checklist

- [x] Unauthenticated requests rejected by the service.
- [x] MFA/TOTP generated and validated.
- [x] RBAC applied with least-privilege permissions.
- [x] Frontend and database placed on separate networks.
- [x] Default-deny firewall used with explicit allow rules.
- [x] Container configured as non-root.
- [x] Container root filesystem made read-only.
- [x] Linux capabilities dropped and `no-new-privileges` enabled.
- [x] Image scanned for HIGH and CRITICAL vulnerabilities.
- [x] Expansion screenshots captured and inserted.

## Cleanup and Teardown

After all evidence has been captured, temporary containers, networks and the kind cluster can be removed:

```bash
docker rm -f authsvc db app web hardened 2>/dev/null
docker network rm frontend-net backend-net 2>/dev/null
kind delete cluster --name ccse-lab4
```

## Conclusion

This lab showed that cloud security requires complementary controls. Authentication verified credentials, MFA reduced the value of a stolen password, and RBAC restricted the actions available to a developer identity. Network segmentation prevented the public web tier from reaching the database directly, while default-deny firewall rules blocked unapproved inbound traffic. Non-root execution, capability dropping, a read-only filesystem, `no-new-privileges` and image scanning reduced the container attack surface. The resulting design follows least privilege across identity, network and compute layers.

## References

- IKB42603 Cloud Computing Security Essentials, Lab 4 manual, UniKL MIIT.
- [Docker Engine Security](https://docs.docker.com/engine/security/)
- [Kubernetes RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- Cloud Security Alliance, *Security Guidance v5*, IAM and Infrastructure/Networking.