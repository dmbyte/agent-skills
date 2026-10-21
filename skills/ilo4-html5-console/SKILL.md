---
name: ilo4-html5-console
description: Operate HPE iLO4 (v2.x) through the owner-scoped HTML5 KVM broker:
  re-establish an expired session, launch the HTML5 console, attach virtual
  media from an HTTP URL, and open Serial-over-LAN through SM-CLP. Includes the
  input-reliability rules that decide which console channel is usable.
---

# HPE iLO4 HTML5 console and SOL

Validated on iLO4 Advanced 2.82 (Feb 06 2023), DL360p Gen8. iLO4 differs from
iLO5 in page structure, key handling and serial plumbing, so do not reuse an
iLO5 recipe.

## Re-establish an expired KVM session

The broker expires a session after ~45 minutes without a client request, and
iLO4 logs its own web session out independently (often after a server power
event). Symptom: every call raises "KVM session is unavailable or expired".

```python
from bmc_html5_kvm import KVMClient
kvm = KVMClient.create("https://<ilo4>", ignore_https_errors=True)  # reuse or new
kvm.select_frame(1)                      # /html/login.html
kvm.fill("#usernameInput", user, confirm=True)
kvm.fill("#passwordInput", password, confirm=True)
kvm.click("#ID_LOGON", confirm=True)
```

Then open the console. The iLO4 UI is a frameset, so select the frame before
every inspect(); `inspect()` on the wrong frame returns no controls:

1. `select_frame(4)` (navigation.html) then `click("#toggleExpandCollapseAll")`
   - the nav groups are collapsed, so "Remote Console" is absent until expanded.
2. `click("#tabset_rc")` (Remote Console).
3. `select_frame(6)` (rc_info.html) then `click("#HRCButton")` - HTML5 console.
4. `inspect()["consoleSurfaces"]` now reports `{"frameIndex": 1, "selector": "#rc_video"}`.
   `select_frame(1)` then `focus_console("#rc_video", confirm=True)`.

Selector limit: the broker rejects selectors longer than 256 characters, and
iLO4 generates 200-280 character CSS paths. Always prefer the `#id` controls
above; the summary page link labelled "HTML5" is only reachable by a 260-char
selector and will fail.
## Virtual media from an HTTP URL

Publish the image with the managed host (never an ad-hoc server on a container
IP), then attach it. Slot 1 is the floppy; iLO4 needs the HP OEM action.

```python
import sys; sys.path.insert(0, "/home/prime/.prime/agent/skills/lan-web-host/scripts")
from prime_web_host import WebHost
url = f"{base}/payload_floppy.img"     # WebHost().static(...) -> file_url(...)
rf("/redfish/v1/Managers/1/VirtualMedia/1/Actions/Oem/Hp/HpiLOVirtualMedia.EjectVirtualMedia/", "POST", {})
rf("/redfish/v1/Managers/1/VirtualMedia/1/Actions/Oem/Hp/HpiLOVirtualMedia.InsertVirtualMedia/", "POST", {"Image": url})
```

- An already-connected slot answers `iLO.0.10.MaxVirtualMediaConnectionEstablished`
  - eject first; URI media stays attached across resets.
- `Inserted: true` on `/redfish/v1/Managers/1/VirtualMedia/1/` is not proof the
  image is readable: iLO4 fetches lazily. Prove it by reading the media from the
  guest (for example GRUB `ls (fd0)`).

## Serial-over-LAN (SOL) needs two switches, not one

1. BIOS side: RBSU (F9 at POST) > "BIOS Serial Console & EMS" >
   "BIOS Serial Console Port" = COM 1 (IRQ4, 3F8h), "Baud Rate" = 115200, F10
   to confirm exit. Without this, SOL is silent no matter what the BMC says.
2. BMC side, over SM-CLP (SSH) - `start /system1/oemhp_vsp1` prints
   "Virtual Serial Port Active: COM1". `set /system1/oemhp_vsp1 enabledstate=enabled`
   is not supported (status=2); attach per session instead.

```
ssh -tt -o ProxyCommand="socat - PROXY:<proxy>:%h:%p,proxyport=<port>" \
    -o KexAlgorithms=diffie-hellman-group-exchange-sha1,diffie-hellman-group14-sha1 \
    -o HostKeyAlgorithms=ssh-rsa -l <user> <ilo4>
start /system1/oemhp_vsp1        # attach; ESC ( returns to the CLP prompt
stop  /system1/oemhp_vsp1        # required after every server reset
```

- After any server reset the tunnel goes stale: `start` then fails with
  "Virtual Serial Port is currently in use by another session." Run
  `stop` then `start`. A dead local ssh that still holds the tunnel must be
  killed first.
- POST text appears on SOL once BIOS redirection is on, so SOL is the trusted
  read/write channel for firmware-level work.
## Linux `console=ttyS0` may not exist even with BIOS redirection on

Booting with console=ttyS0,115200n8 produced no serial output, and with that
as the only console the kernel panicked ("Failed to acquire terminal",
"Attempted to kill init! Comm: bash") - the 8250 port was not registered, so
BIOS-level redirection working does not prove an OS serial console. Confirm the
OS really attaches before relying on it; otherwise use a boot-time payload.

## Catching the GRUB menu without guessing timeouts

The GRUB highlight is green, so detect it by pixel fraction instead of timing.
Keep sending Escape so the menu does not time out; re-arm after each capture.

```python
import numpy as np; from PIL import Image
def grub_green(fn, box=(200,700,330,1000)):
    a = np.asarray(Image.open(fn).convert("RGB"), dtype=np.int16)[box[0]:box[1], box[2]:box[3]]
    r, g, b = a[...,0], a[...,1], a[...,2]
    return float(((g > 110) & (g > r + 25) & (g > b + 25)).mean())
# after ForceRestart: loop { 5x key("Escape"), capture, if grub_green() > 0.01: menu is up }
```

A stale video frame is not proof of state: compare two captures (pixel diff)
before believing the console is idle.
