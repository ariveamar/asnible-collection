# Ansible — Kill & Start Services

Stop multiple Linux services by killing PIDs (via `pgrep`), with full validation and output.  
Start services using a provided shell script (`/home/start-service.sh`).

---

## Directory Structure

```
ansible-kill-services/
├── stop_services.yml                         ← Stop-only playbook
├── start_services.yml                        ← Start-only playbook
├── restart_services.yml                      ← Stop + Start playbook
├── inventory/
│   └── hosts.ini                             ← Target host inventory
└── roles/
    └── kill_services/
        ├── defaults/main.yml                 ← All variables with defaults
        ├── meta/main.yml                     ← Role metadata
        └── tasks/
            ├── main.yml                      ← Entry point + loop
            ├── stop_service.yml              ← Kill PID logic + validation
            └── start_service.yml             ← Start script logic + validation
```

---

## Variables

### Target Host
| Variable           | Default         | Description                                |
|--------------------|-----------------|--------------------------------------------|
| `target_host_ip`   | `192.168.1.10`  | IP of the server where PIDs will be killed |

### Services List
```yaml
services_to_stop:
  - name: "nginx"              # Process name used with pgrep -f
    kill_signal: "TERM"        # TERM (graceful) or KILL (force)
    force_kill_enabled: true   # Send SIGKILL if SIGTERM doesn't stop it
```

### Timing
| Variable                | Default | Description                                   |
|-------------------------|---------|-----------------------------------------------|
| `stop_wait_seconds`     | `5`     | Wait after SIGTERM before re-check             |
| `force_kill_wait_seconds` | `3`   | Wait after SIGKILL before final check          |
| `max_retry_attempts`    | `3`     | Retry count for stop validation                |
| `retry_delay_seconds`   | `3`     | Delay between retries                          |
| `start_wait_seconds`    | `10`    | Wait after start script before PID check       |

---

## Stop Flow (per service)

```
STEP 1 → pgrep -f <name>            Find initial PIDs
STEP 2 → kill -TERM <PIDs>          Graceful shutdown
STEP 3 → Wait N seconds             Grace period
STEP 4 → pgrep -f <name>            Re-check (still running?)
STEP 5 → kill -9 <PIDs>             Force kill if still alive (if enabled)
STEP 6 → Retry validation (3x)      Confirm truly stopped
STEP 7 → Print final PID status     Show "No PID found" output
STEP 8 → Assert pass/fail           Fail play if still running
```

---

## Usage

```bash
# Stop all services
ansible-playbook -i inventory/hosts.ini stop_services.yml

# Start all services
ansible-playbook -i inventory/hosts.ini start_services.yml

# Restart (stop then start)
ansible-playbook -i inventory/hosts.ini restart_services.yml

# Override target IP at runtime
ansible-playbook -i inventory/hosts.ini stop_services.yml \
  -e "target_host_ip=10.0.0.50"

# Dry run (check mode)
ansible-playbook -i inventory/hosts.ini stop_services.yml --check

# Add verbosity for debug
ansible-playbook -i inventory/hosts.ini stop_services.yml -v
```

---

## Sample Output

```
TASK [STEP 1 — Find running PIDs]
ok: [192.168.1.10]

TASK [STEP 1 — Show initial PID status]
ok: [192.168.1.10] =>
  msg:
    Service  : nginx
    Status   : RUNNING
    PID(s)   : 1234, 1235

TASK [STEP 2 — Send SIGTERM]
changed: [192.168.1.10]

TASK [STEP 7 — Final PID status output]
ok: [192.168.1.10] =>
  msg:
    STATUS   : ✅ STOPPED — No PID found for 'nginx'
    PID(s)   : None

TASK [STEP 8 — ASSERT]
ok: [192.168.1.10] => ✅ PASSED: Service 'nginx' is confirmed STOPPED
```
