## 1) Fix for parsing the OpenVPN version

**Problem (symptoms):**

In the monitor log you see an error like:

```
ValueError: 2.7-0.20250326gitd167815.el10_0 is not valid SemVer string
```

This happens because OpenVPN from some Linux distributions returns a version string with “suffixes” (e.g., `2.7-0.2025…el10_0`), while the `semver` library only accepts clean `MAJOR.MINOR[.PATCH]` formats.

**What changes:**

- Instead of taking the “second word” from the banner and passing it directly to `semver`, we first extract only the core version: `MAJOR.MINOR[.PATCH]` (e.g., `2.7` → converted to `2.7.0`).
- If nothing valid is found, we fall back to a safe value `0.0.0`.

**Why:**

- Ensures `semver` always receives a valid string.
- Prevents crashes.
- Keeps the monitor working even with distribution-specific suffixes.

**Where it happens:**

- File: `openvpn_monitor/vpns/openvpn/data_collector.py`  
  In your case:  
  `/usr/local/lib/python3.12/site-packages/openvpn_monitor/vpns/openvpn/data_collector.py`  
  (Other installs: `/usr/lib/python3.12/site-packages/...`)

**When to apply:**

- Any time you see the `ValueError` above with a version like `2.7-0.20…`.
- Whenever OpenVPN returns a version string with a distro suffix (`-0.`, `el9`, `el10`, etc.).

---

## 2) Fix for parsing the “remote” address from management interface (status 3)

**Problem (symptoms):**

In the monitor log you see an error like:

```
ValueError: 'udp4:80.238.124.120:53932' does not appear to be an IPv4 or IPv6 address
```

OpenVPN 2.7 can return the `remote` field as `udp4:IP:PORT` or `tcp6:[IPv6]:PORT`.

The old code tried to pass this directly to `ip_address(remote)`, which fails.

**What changes:**

- Added a helper `_extract_host_port(remote)` that:
  - Removes `udp`/`tcp` prefixes with optional `4`/`6` (e.g., `udp4:`).
  - Supports IPv6 in brackets `[ ]`.
  - Splits into host and port (if present).
  - Returns `(host, port)`.
- In `parse_status()`, we now:
  - Call the helper first.
  - Wrap `ip_address(host)` in `try/except`.
  - If the host can’t be parsed, store an empty address/port instead of crashing.

**Why:**

- Allows the monitor to handle both classic IP and new formats like `udp4:IP:PORT` / `tcp6:[IPv6]:PORT` without exceptions.

**Where it happens:**

- Same file: `openvpn_monitor/vpns/openvpn/data_collector.py`, in the `parse_status()` method, where it previously did:
  ```python
  remote_ip = ip_address(remote)
  ```

**When to apply:**

- Whenever you see the above `ValueError` for `udp4:…` / `tcp6:…`.
- Proactively, if you use OpenVPN 2.7 and the status 3 output contains `remote` fields in these newer formats.

---

### Summary in one sentence

- **Version fix:** sanitize the version string so `semver` gets a proper `MAJOR.MINOR[.PATCH]`.  
- **Address fix:** normalize `remote` fields like `udp4:IP:PORT` or `tcp6:[IPv6]:PORT` before passing them to `ip_address()`.
