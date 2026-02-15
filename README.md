# SafeSetID Policy Manager

This repository contains the `safesetid` script — a policy manager for the SafeSetID Linux Security Module (LSM).

The script writes base policies to the SafeSetID sysfs interface, detects newly created users/UIDs, and can restrict them. It features a robust self-healing model (wrapper, hardened backups, `tmpfs` runtime store) to ensure the integrity of the system.

Important: This is a Bash script and requires kernel support for SafeSetID exposed at `/sys/kernel/security/safesetid`. If this directory is missing, no kernel policies will be modified.

## Why Use SafeSetID? A Comparison with `NoNewPrivileges`

Before using this tool, it's helpful to understand what SafeSetID provides compared to other security mechanisms like systemd's `NoNewPrivileges=yes`.

In short:
- `NoNewPrivileges=yes` is a hammer. It's a one-way switch that prevents a process and all of its children from gaining new privileges (e.g., from setuid binaries). It's powerful but coarse.
- SafeSetID is a scalpel. It's a rule-based LSM that lets you define exactly which UID/GID identity transitions are allowed. It doesn’t block all privilege transitions, only those not explicitly on the allow-list.

### Feature Comparison

| Feature | `NoNewPrivileges=yes` (systemd) | SafeSetID (Kernel LSM) |
| :--- | :--- | :--- |
| **Goal** | Prevent any future privilege gains in a process tree. | Control identity transitions by explicitly allow-listing UID/GID changes. |
| **Granularity** | Coarse (global on/off per process tree). | Fine-grained (per-transition rules). |
| **Flexibility** | Low—can break legitimate workflows. | High—allows specific transitions while blocking unexpected ones. |
| **Use Case** | Long-running daemons that never need privilege changes. | Tools or workflows that require specific, controlled transitions. |
| **Operational Complexity** | Easy to enable, but blunt for complex apps. | Requires policy management (which this script automates). |
| **Recovery / Self-Healing** | None. | **Supported by this project** (wrapper, backups, tmpfs, monitors). |

This `safesetid` script acts as a policy manager, making the powerful SafeSetID kernel feature practical and automated.

---

## Current Status and Behavior

- Oneshoot and path-triggered units:
  - `safesetid.service` is `Type=oneshot`. It may appear `inactive` after successful execution (`Result=success`).
  - `safesetid-monitor.service` and `safesetid-file-monitor.service` are triggered by their respective `.path` units and are typically `static/inactive` until a file change occurs.
- Self-healing:
  - Hardened backups are kept in `/var/lib/safesetid` (optionally immutable).
  - A runtime trust anchor (`tmpfs` at `/var/tmp/safesetid`) holds authoritative copies; monitors and verify repair deviations.
- Internal commands (protected):
  - Internal maintenance commands (prefixed with `_` and `check_function`) cannot be run directly from a terminal and are executed only via the wrapper or systemd.
- DoS mitigation:
  - High-frequency events are tracked; a mitigator may be scheduled to reduce load and perform conservative repairs.
- Locks:
  - Global locking avoids concurrent modifications; if `flock` is unavailable, the script falls back to best-effort temporary locks.

---

## Installation & Administration

### Install or Update
The installer is idempotent and sets up all components, backups, hashes, and systemd units.
```bash
sudo ./safesetid install_script
```
During installation a 32-bit uninstall token (hex, 8 chars) is generated and stored at `/var/lib/safesetid/.uninstall_token`. The installer logs the token once — store it securely.

### Configure Base Policies
Edit and apply UID/GID policies. Must be invoked via `sudo`/`doas` (not from a direct root shell).
```bash
sudo safesetid configure_uid
sudo safesetid configure_gid
```

### Manual Verification
Run integrity checks and self-healing. Typically triggered automatically by `.path` units.
```bash
sudo safesetid verify
```

### Uninstall (Protected)
Uninstall requires:
1. Invocation via `sudo`/`doas` from a regular user.
2. The uninstall token as the second argument (generated at install, stored in `/var/lib/safesetid/.uninstall_token`).
3. Interactive confirmation.
```bash
sudo safesetid uninstall_script <UNINSTALL_TOKEN>
```

---

## Systemd Units

- `var-tmp-safesetid.mount`: mounts the runtime `tmpfs`.
- `safesetid.service`: oneshot boot-time application of policies.
- `safesetid-wrapper-restore.path` + `.service`: restores the wrapper if changed/deleted.
- `safesetid-monitor.path` + `.service`: watches `/etc/passwd` and runs a UID check.
- `safesetid-file-monitor.path` + `.service`: watches critical script/config files and triggers verification.

Note: Path-triggered services are expected to be `inactive/static` and only run on file events. Oneshoot `safesetid.service` is `inactive` after successful completion.

---

## Smoke Test

A non-destructive smoke test is provided under `Test/smoke_test`. It:
- Runs syntax and ShellCheck.
- Invokes `verify` and validates datastore integrity and wrapper backups.
- Tests lock behavior by running verify and check_function concurrently (check_function is blocked from terminal, as designed).
- Checks all systemd units, treating oneshot/static/path-triggered units correctly.
- Optionally triggers `.path` units via `touch /etc/passwd` and `touch /etc/group` and prints recent journal entries.

Run:
```bash
sudo ./Test/smoke_test
```

---

## Troubleshooting

- View unit status and logs:
```bash
sudo systemctl status safesetid-*.path safesetid-*.service var-tmp-safesetid.mount
sudo journalctl -u safesetid.service -n 50 --no-pager
sudo journalctl -u safesetid-file-monitor.service -n 50 --no-pager
```
- If `/sys/kernel/security/safesetid` is missing, kernel support is not available; the script will skip kernel policy writes.

---

## Security Notes

- This is not a high-security product; it’s a practical policy manager with bonus self-protection.
- Self-healing and immutable backups raise the bar against accidental changes and simple tampering. A local root attacker can still disable protections.
- For stronger guarantees, consider Secure Boot, IMA/EVM, and remote/WORM logging.

---

## License

MIT License
