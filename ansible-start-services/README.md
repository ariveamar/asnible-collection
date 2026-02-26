# Ansible — Start Services via Shell Scripts

Start multiple Linux services by executing shell scripts from defined directory paths on a target server.  
Full STDOUT/STDERR capture, per-script validation, looping over multiple directory paths, and configurable IP targeting.

---

## Directory Structure

```
ansible-start-services/
├── start_services.yml                              ← Main playbook
├── inventory/
│   └── hosts.ini                                   ← Target host inventory
└── roles/
    └── start_services/
        ├── defaults/main.yml                       ← All variables and defaults
        ├── meta/main.yml                           ← Role metadata
        └── tasks/
            ├── main.yml                            ← Entry point + loop
            └── start_service.yml                   ← Core: validate + execute + output + verify
```

---

## Variables

### Target Server
| Variable          | Default         | Description                                        |
|-------------------|-----------------|----------------------------------------------------|
| `target_host_ip`  | `192.168.1.10`  | IP of the server where scripts will be executed    |

### Start Script List
```yaml
start_scripts:
  - label:       "My Service"              # Friendly name shown in all output
    script_dir:  "/opt/myapp"              # Directory path on the target host
    script_file: "start.sh"               # Script filename to execute
    args:        "--env production"        # Optional: arguments to pass to script
```

### Execution Settings
| Variable               | Default | Description                                              |
|------------------------|---------|----------------------------------------------------------|
| `start_wait_seconds`   | `10`    | Seconds to wait after script runs (service init time)    |
| `script_timeout`       | `120`   | Max seconds before script is considered timed out        |
| `ignore_script_errors` | `false` | If true, continue loop even if a script returns rc != 0  |

---

## Execution Flow (per script entry in loop)

```
BANNER   → Print service label, target IP, full script path
STEP 1   → stat <script_dir>                 Verify directory exists on target
STEP 2   → stat <script_dir>/<script_file>   Verify script file exists
STEP 3   → chmod 0755 <script>               Ensure script is executable
STEP 4   → ./script.sh <args>  (chdir)       Execute script from its own directory
STEP 5   → Show full STDOUT                  Print every line the script output
STEP 6   → Show STDERR (if any)             Print any error output from the script
STEP 7   → assert rc == 0                   Fail play if script returned non-zero
STEP 8   → pause N seconds                  Wait for service to initialise
STEP 9   → ps aux | grep <process>          Check if process is now running
STEP 10  → Print final status output        Show PID(s) or "not detected" with note
```

---

## Usage

```bash
# Start all services with default IP
ansible-playbook -i inventory/hosts.ini start_services.yml

# Override target IP at runtime
ansible-playbook -i inventory/hosts.ini start_services.yml \
  -e "target_host_ip=10.0.0.50"

# Continue loop even if one script fails
ansible-playbook -i inventory/hosts.ini start_services.yml \
  -e "ignore_script_errors=true"

# Dry run (validates without executing)
ansible-playbook -i inventory/hosts.ini start_services.yml --check

# Verbose mode
ansible-playbook -i inventory/hosts.ini start_services.yml -v
```

---

## Sample Output

```
TASK [[ APP SERVICE A ] STEP 5 — Script STDOUT output]
ok: [192.168.1.10] =>
  msg:
    ┌─────────────────────────── STDOUT ────────────────────────────┐
      Service   : App Service A
      Script    : /opt/services/app-a/start.sh
      Target IP : 192.168.1.10
      Exit Code : 0
      Status    : ✅ SUCCESS (rc=0)
    ├───────────────────────────────────────────────────────────────┤
      Starting App Service A...
      Loading config from /opt/services/app-a/config.yml
      Binding to port 8080...
      Service started successfully. PID: 4821
    └───────────────────────────────────────────────────────────────┘

TASK [[ APP SERVICE A ] STEP 10 — Final process verification output]
ok: [192.168.1.10] =>
  msg:
    === Process Status for service: App Service A ===
    Timestamp  : 2025-06-01 10:32:45
    Host IP    : 192.168.1.10
    Script     : /opt/services/app-a/start.sh

    STATUS     : ✅ RUNNING — Process found for 'App Service A'
    PID(s)     : 4821
    =============================================================
```

---

## Notes

- Scripts are executed with `chdir` set to their directory, so relative paths inside scripts work correctly.
- The `$$` self-exclusion bug (common in Ansible shell tasks) is avoided by using a `ps aux | grep -w | grep -v` pipeline instead of `pgrep`.
- All stdout output lines are captured and displayed individually via `stdout_lines`.
- If `ignore_script_errors: true`, the loop continues even if a script fails — useful for partial restarts.
