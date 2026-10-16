# Vulnserver Buffer Overflow Exploit Development

A walkthrough of developing a remote code execution exploit against
[Vulnserver](https://github.com/stephenbradshaw/vulnserver) (the `TRUN`
command) using **Immunity Debugger** on Windows and **Python 3** on Ubuntu.

Vulnserver is a classic, intentionally-vulnerable training target used
across OSCP-style exploit-development courses — safe to practice on
because it's built to be broken.

## Lab setup

| Machine         | Role                                              | IP               |
|------------------|----------------------------------------------------|------------------|
| Windows VM/host  | Runs Vulnserver.exe, Immunity Debugger attached     | 192.168.137.1    |
| Ubuntu VM        | Runs the Python fuzzer, netcat                      | 192.168.137.x    |

**Tools used:**
- [Immunity Debugger](https://www.immunityinc.com/products/debugger/) (Windows)
- [Vulnserver](https://github.com/stephenbradshaw/vulnserver) (Windows, target)
- [mona.py](https://github.com/corelan/mona) (Immunity PyCommands plugin)
- Python 3 + `netcat` (Ubuntu)

## Progress log

### 1. Confirm connectivity to the target
Used `netcat` from Ubuntu to confirm the Windows target was reachable
on port 9999 before fuzzing:
```bash
nc -nv 192.168.137.1 9999
```
![netcat connect](screenshots/02_netcat_connect.png)

Vulnserver's own console confirmed the inbound connection:
![vulnserver listening](screenshots/03_vulnserver_listening.png)

### 2. Attach Immunity Debugger to Vulnserver
Opened `vulnserver.exe` in Immunity Debugger (File → Open), ran it with
F9. Immunity sits paused at the module entry point until the process
is actually exercised:
![Immunity attached](screenshots/01_immunity_attached.png)

### 3. Write the fuzzer
Wrote `exploit/1_fuzzer.py` on the Ubuntu VM in `nano` — sends the
`TRUN` command to Vulnserver with a buffer of `"A"` characters that
grows by 100 bytes each iteration until the connection drops:
![fuzzer script in nano](screenshots/04_fuzzer_script.png)
![fuzzer.py created](screenshots/05_fuzzer_created.png)

Hit (and fixed) a syntax error on the first run — the shebang line
(`#!/usr/bin/env python3`) lost its leading `#` during copy/paste:
![syntax error](screenshots/07_syntax_error_fixed.png)

### 4. Run the fuzzer
```bash
python3 fuzzer.py
```
![running fuzzer](screenshots/06_running_fuzzer.png)
![fuzzing progress](screenshots/08_fuzzing_progress.png)

Buffer size climbed steadily (100, 200, 300 ... bytes) with each
successful send printed to the terminal.

### 5. Crash observed
Vulnserver stopped responding and Immunity Debugger caught the fault.
At this point in the run:
![crash in Immunity](screenshots/09_crash_eip.png)

- **EIP:** `7731850C` — inside `ntdll`, not a direct overwrite with
  `41414141` yet
- **Status bar:** `Terminated`, exit code 1

**What this means:** the overflow crashed the process, but the crash
is currently happening inside a Windows system DLL rather than landing
directly in EIP with controlled bytes. This is a common checkpoint —
it usually means either the buffer needs to be larger before it reaches
EIP directly, or this particular crash path is going through SEH
instead of a direct return-address overwrite. Both are solvable; this
is the next thing to run down before moving to the offset-finding stage
with `mona pattern_create` / `pattern_offset`.

## Next steps
- [ ] Re-run with a larger max buffer size and confirm whether EIP
      lands on a clean `41414141` instead of an `ntdll` address
- [ ] If it's consistently landing in `ntdll`/SEH, check the SEH chain
      in Immunity for an overwritten handler
- [ ] Once EIP (or SEH) control is confirmed, generate a cyclic pattern
      with `!mona pattern_create 3000` and find the exact offset
- [ ] Bad character analysis
- [ ] Find a JMP ESP / POP POP RET gadget
- [ ] Generate and deliver shellcode

## Disclaimer

For educational use only, against intentionally vulnerable software
(Vulnserver) running in an isolated lab environment that I own and
control. Do not use against systems you don't have explicit
authorization to test.

## References
- Vulnserver: https://github.com/stephenbradshaw/vulnserver
- mona.py: https://github.com/corelan/mona
- Corelan's exploit-writing tutorial series (the original reference for this methodology)
