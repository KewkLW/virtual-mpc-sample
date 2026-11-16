# Linux

How to run the lab on Linux, and what has been checked there. The steps are those of the
[README](../README.md); this page has the parts that differ from macOS.

## What was run

One machine, 2026-11-15 and 2026-11-16: x86_64 with 32 threads, an Arch-based distribution,
QEMU 11.1, Docker 29, Python 3.15, firmware 1.3.0.12. From a fresh clone, in this order:
`extract`, `build`, `run --headless`, the panel page with the screen, a pad hit that gives
sound, `ssh`, `autorun` with an SD package (the card goes in while the VM runs), `autorun
--results`, `stop`.

Measured in that run, with other work on the host (load average about 30):

| | |
|---|---|
| `build`, first time (kernel), measured the day before with the same build scripts | about 8 minutes |
| `build`, tools image and kernel already built | 1 min 12 s |
| `run` until `lab.py ssh` answers | 31 s |
| `run` until the app's audio runs | 39 s |
| the stock screen | there when looked at after 90 s |
| `autorun` with a small package (card, power cycle, script) | 1 min 46 s |
| `stop` | 13 s |

The update used was inMusic's updater download for macOS, a `.zip` with `Update.img`
inside; `extract` reads it as it is. The updater for Windows (an `.exe`) was not tried.

Not run: Linux on arm64 (KVM), other distributions, a host with SELinux, rootless Docker,
Podman. The package names for Debian and Fedora below come from those distributions'
package lists, not from a run.

## Packages

| | Arch | Debian, Ubuntu | Fedora |
|---|---|---|---|
| QEMU | `qemu-system-aarch64 qemu-hw-display-virtio-gpu qemu-hw-display-virtio-gpu-pci qemu-img` | `qemu-system-arm qemu-utils` | `qemu-system-aarch64 qemu-device-display-virtio-gpu qemu-device-display-virtio-gpu-pci qemu-img` |
| Docker (only for `build`) | `docker` | `docker.io` | `moby-engine`, or Docker's own packages |
| Others | `python openssh` | `python3 openssh-client` | `python3 openssh-clients` |

Some distributions ship each QEMU device model as its own package. `lab.py run` checks the
installed QEMU first and names the device that is missing, for example:

```
lab: this QEMU has no virtio-gpu-pci: install the package(s) with these device models.
```

Your user must be allowed to use Docker (member of the `docker` group, or rootless Docker).
`lab.py` calls `docker` itself, without `sudo`.

## Start

```bash
python3 lab.py extract PATH/TO/UPDATE   # the update image, or the .zip that holds it
python3 lab.py build                    # kernel (about 8 minutes on 32 threads), bridge, images
python3 lab.py run --headless           # the VM and the panel server
```

Open http://127.0.0.1:8765/ and click the page once for sound.

`--headless` is the usual way on Linux: the panel page shows the screen. Without it QEMU
opens its own window if one of its display packages (GTK or SDL) is installed. Without
those packages `python3 lab.py run` works the same and no window appears (seen here).

## What differs from macOS

**Software emulation on x86.** A PC has no way to run the ARM guest directly, so QEMU
emulates the CPU (TCG). It is slower than the device and it needs steady CPU time: on a
busy host the guest falls behind real time and the sound breaks up. On arm64 Linux with
`/dev/kvm`, `lab.py` uses KVM instead (not run).

**Audio buffer.** Under TCG the lab gives the app's audio device 4096 frames (93 ms)
instead of 1024. With 1024 on a busy host the app's audio stream failed for good
(`Unexpected audio capture error: -5` in the app's log) and the next Load Project or New
Project never finished. 4096 makes that rarer, not impossible: it still happened once at a
load average near 90 on 32 threads, and in a run of several hours with QEMU at a higher
priority (`nice -10`) the stream reported three errors and went on each time. For the
smaller buffer: `python3 lab.py run --extra snd_aloop.min_buffer=1024` (the later option
wins).

**Files that Docker writes.** Docker's daemon runs the build container as root, so `out/`
would belong to root and `lab.py ssh` could not read its own key. After every build
`lab.py` gives `out/` back to your user. With rootless Docker the files are yours already
and `lab.py` leaves them alone (not run).

**SELinux.** With SELinux on, `lab.py` mounts the repository into the build container with
the `z` option, so that the container may read and write it (not run). This relabels the
repository folder; keep the clone in a folder of its own.

**The SD slot on QEMU 11.** QEMU 11 treats an SD card slot that starts empty as not
removable, and a card could then never be inserted (`Device 'sd0' is not removable`).
`lab.py run` now starts an empty slot with a blank placeholder and takes it out before the
guest runs. Inserting a card later works again: tried from the panel (insert, the guest
mounts it, eject) and with `lab.py autorun`.

## If something does not work

| What you see | What to do |
|---|---|
| `qemu-system-aarch64 not found in PATH` | Install the QEMU packages above. |
| `this QEMU has no ...` | Install the package with that device model (table above). |
| `permission denied` from `docker` | Add your user to the `docker` group and log in again, or use rootless Docker. |
| `out/keys/id_ed25519`: `Permission denied` on `lab.py ssh` | Files from a build before this fix: `python3 lab.py build bridge` gives `out/` back. |
| No sound in the page | Click the page once. Only one page gets the sound: close other tabs with the panel and reload. |
| Sound breaks up, the panel reacts late | The host is busy. Close what uses the CPU, or give QEMU a higher priority. |
| "Loading Project" does not end | The app lost its audio device (see Audio buffer). `python3 lab.py stop`, then `run` again. |
