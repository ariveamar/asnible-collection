# Ansible — Stop & Start Services via Shell Scripts

Execute stop/start shell scripts from defined directory locations on a target Linux server.  
Full STDOUT capture, per-script validation, looping over multiple paths, and configurable IP targeting.

---

## Directory Structure

```
ansible-script-services/
├── stop_services.yml                             ← Stop-only playbook
├── start_services.yml                            ← Start-only playbook
├── restart_services.yml                          ← Stop then Start playbook
├── inventory/
│   └── hosts.ini                                 ← Server inventory with IPs
└── roles/
    └── script_services/
        ├── defaults/main.yml                     ← All variables & defaults
        ├── meta/main.yml                         ← Role metadata
        └── tasks/
            ├── main.yml                          ← Stop loop entry point
            ├── start_main.yml                    ← Start loop entry point
            └── execute_script.yml                ← Core: validate + run + output
```

---

## Variables

### Target Server
| Variable          | Default          | Description                                      |
|-------------------|------------------|--------------------------------------------------|
| `target_host_ip`  | `192.168.1.10`   | IP of the server where scripts will be executed  |

### Script List Structure
```yaml
stop_scripts:           # or start_scripts
  - label:       "My Service"            # Human-readable name in output
    script_dir:  "/opt/services/myapp"  # Directory path on the target host
    script_file: "stop.sh"              # Script filename to execute
    args:        "--graceful"           # (Optional) arguments passed to the script
```

### Execution Settings
| Variable               | Default | Description                                          |
|------------------------|---------|------------------------------------------------------|
| `stop_wait_seconds`    | `5`     | Pause after each stop script completes               |
| `start_wait_seconds`   | `10`    | Pause after each start script completes              |
| `script_timeout`       | `120`   | Max seconds before a script is considered timed out  |
| `ignore_script_errors` | `false` | If true, continue loop even if a script fails (rc≠0) |

---

## Execution Flow (per script in loop)

```
STEP 1 → stat <script_dir>             Verify directory exists
STEP 2 → stat <script_dir>/<file>      Verify script file exists
STEP 3 → chmod 0755 <script>           Ensure script is executable
STEP 4 → shell: ./<script> <args>      Execute script (chdir to script_dir)
STEP 5 → debug: stdout output          Print full STDOUT from script
STEP 6 → debug: stderr output          Print STDERR if any errors
STEP 7 → assert: rc == 0              Fail play if script returned non-zero
STEP 8 → pause N seconds              Wait before processing next service
```

---

## Usage

```bash
# Stop all services
ansible-playbook -i inventory/hosts.ini stop_services.yml

# Start all services
ansible-playbook -i inventory/hosts.ini start_services.yml

# Full restart (stop then start)
ansible-playbook -i inventory/hosts.ini restart_services.yml

# Override target IP at runtime
ansible-playbook -i inventory/hosts.ini stop_services.yml \
  -e "target_host_ip=10.0.0.50"

# Continue even if one script fails
ansible-playbook -i inventory/hosts.ini stop_services.yml \
  -e "ignore_script_errors=true"

# Dry run
ansible-playbook -i inventory/hosts.ini stop_services.yml --check

# Verbose output
ansible-playbook -i inventory/hosts.ini stop_services.yml -v
```

---

## Sample Output

```
TASK [[ APP SERVICE A ] STEP 5 — Script STDOUT output]
ok: [192.168.1.10] =>
  msg:
    ┌───────────────────── STDOUT ─────────────────────────────────
      Script    : /opt/services/app-a/stop.sh
      Target IP : 192.168.1.10
      Exit Code : 0
      Status    : ✅ SUCCESS
    ├───────────────────── OUTPUT ─────────────────────────────────
      Stopping App Service A...
      Sending SIGTERM to PID 4521...
      Service stopped successfully.
    └──────────────────────────────────────────────────────────────

TASK [[ APP SERVICE A ] STEP 7 — Assert script completed successfully]
ok: [192.168.1.10] => ✅ PASSED: Script completed successfully (rc=0)
```
