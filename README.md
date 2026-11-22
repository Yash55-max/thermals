# Thermals

[![CI](https://github.com/Yash55-max/thermals/actions/workflows/ci.yml/badge.svg)](https://github.com/Yash55-max/thermals/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![Linux: All Major Distros](https://img.shields.io/badge/platform-Linux-orange.svg)](https://kernel.org)

**Thermals** (`asus-thermals-control`) is a lightweight, zero-dependency Linux hardware telemetry monitor and platform thermal controller. Designed for modern laptops and desktop workstations, Thermals provides real-time sensor monitoring, a closed-loop graduated thermal limiter, battery longevity controls, CPU Turbo Boost regulation, and dual graphical (GUI) and command-line (CLI/JSON) interfaces.

---

## Hardware Compatibility Matrix

Unlike generic scripts that make unsupported claims across untested architectures, Thermals strictly distinguishes between hardware verified on physical machines, standard ACPI interfaces expected to work across modern Linux kernels, and vendor-specific features:

| Hardware Category | Subsystem / Driver | Tested & Verified | Expected (Standard ACPI) | Notes |
| :--- | :--- | :---: | :---: | :--- |
| **ASUS Laptops** | `asus-nb-wmi`, `asus_wmi` | **ASUS Vivobook Go E1504FA** | Zenbook, Vivobook, TUF | Native profile switching & battery threshold |
| **AMD Processors** | `k10temp`, `amd_energy` | **Ryzen 5 7520U** | Ryzen 3000/5000/7000/8000 | Tctl, Tdie, per-CCD temps, CPB Boost toggle |
| **Intel Processors** | `coretemp`, `intel_pstate` | — | Core 6th–14th Gen, Xeon | Package, per-core temps, turbo toggle |
| **AMD Graphics** | `amdgpu`, `radeon` | **Radeon 610M (iGPU)** | RX 5000/6000/7000, Radeon Vega | Edge, junction, VRAM telemetry |
| **NVIDIA Graphics** | `nvidia-smi`, `nouveau` | — | GeForce RTX, GTX, Quadro | GPU temp, fan speed %, power draw via SMI |
| **Intel Graphics** | `i915`, `xe` | — | Iris Xe, Arc Alchemist | Kernel hwmon thermal reporting |
| **Storage (NVMe)** | `nvme` kernel driver | **Kingston / Micron NVMe** | Standard NVMe 1.3/1.4/2.0 | Composite temp, sensor 1/2, critical limits |
| **Battery Threshold** | `charge_control_end_threshold` | **ASUS BAT0** | ASUS, Lenovo ThinkPad, modern laptops | 50%–100% hardware charge limiter |
| **Cooling Fans** | `hwmon*/pwm*` | **ASUS WMI Fan** | Standard hwmon PWM chips | Governed by safety rails (minimum 50 PWM floor) |
| **Platform Profiles** | `acpi/platform_profile` | **ACPI Standard** | Modern Dell, Lenovo, HP, ASUS | Quiet, Balanced, Performance modes |

> Community reports are actively welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) to submit telemetry from your hardware model.

---

## Key Features

### 1. Graduated Thermal Limiter (Closed-Loop Thermostat)
Prevents thermal throttling and eliminates profile flapping under bursty workloads. Rather than crude binary switching, the limiter employs a 4-stage graduated regulation pipeline protected by a 30-second minimum dwell timer and 5°C hysteresis:
- **Stage 0 (Normal)**: User's baseline profile and full CPU Turbo Boost.
- **Stage 1 ($T \ge T_{\text{target}}$)**: Scales CPU Energy-Performance Preference (EPP) to `balance_power`.
- **Stage 2 ($T \ge T_{\text{target}} + 3^\circ\text{C}$)**: Temporarily disables CPU Turbo Boost to cap thermal spikes.
- **Stage 3 ($T \ge T_{\text{target}} + 6^\circ\text{C}$)**: Engages Quiet platform profile and maximum cooling acoustics until temperatures stabilize.
- **Headless Daemon**: Can run 24/7 as a background systemd user service (`thermals.service`) without opening the GUI.

### 2. Verified Thermal Performance
On an ASUS Vivobook Go E1504FA (Ryzen 5 7520U) running continuous multi-threaded compilation workloads:
- **Default Performance Profile**: Peaks at **84.5°C** with high acoustic whine.
- **Thermals Active Limiter (Target 72°C)**: Stabilizes peak thermals at **68°C–71°C** (**~15°C–16°C reduction**), maintaining 92% sustained compute throughput with virtually silent fan operation.

### 3. Battery Longevity Limit
Lithium-ion laptop batteries degrade quickly when continuously held at 100% capacity under heat. Thermals enables hardware-level charge capping (`charge_control_end_threshold`) to **60%**, **80%**, or **100%** on ASUS and compatible laptops.

### 4. Fan Safety Rails & Emergency Overrides
Manual fan sliders without safeguards are hazardous to hardware. Thermals enforces:
- **Motor Stalling Safety Floor**: Strictly rejects 0% and clamps all manual PWM requests to at least 50 PWM (~20% duty cycle) to ensure bearings spin and airflow never stops.
- **Emergency Thermal Override**: If hardware reaches **80.0°C** while under manual fan control, Thermals instantly revokes manual mode and restores fans to maximum automatic firmware curves.
- **Exit & Signal Restoration**: Registers `SIGINT`, `SIGTERM`, `SIGHUP`, and `atexit` handlers to guarantee firmware fan control is restored on exit, crash, or reboot.

### 5. Throttling Detection & Resource Attribution
- Real-time detection of thermal throttling by monitoring CPU frequency suppression under elevated temperatures.
- Live process attribution showing the top CPU-consuming processes directly alongside temperature graphs.

---

## Installation

### Method 1: Universal Installer (Recommended)

```bash
git clone https://github.com/Yash55-max/thermals.git
cd thermals

# Install dependencies (Debian/Ubuntu, Fedora, Arch, openSUSE)
bash install.sh --install-deps

# Install for current user (no root required)
bash install.sh --user

# Optional: Set up privileged helper for battery & fan limits
bash install.sh --with-helper

# Optional: Enable background thermal limiter service
bash install.sh --enable-service
```

### Method 2: Distribution Packages

#### Debian / Ubuntu / Linux Mint (.deb)
```bash
# Build package locally
bash package.sh

# Install package
sudo apt install ./dist/asus-thermals-control_1.2.0_all.deb
```

#### Arch Linux / Manjaro (PKGBUILD)
```bash
makepkg -si
```

#### Fedora / RHEL (RPM)
```bash
rpmbuild -bb asus-thermals-control.spec
```

---

## CLI & Headless Usage

Thermals is fully usable in headless environments, remote SSH sessions, or custom desktop status bars (Polybar, Waybar, i3blocks):

```bash
# Formatted hardware overview and status
thermals --status

# Full inventory of all detected hardware sensors and thresholds
thermals --list-hardware

# Machine-readable JSON output (ideal for Waybar/Polybar scripts)
thermals --json

# Interactive real-time terminal dashboard
thermals --monitor

# Set target temperature limit in Celsius (or 'off' to disable)
thermals --set-target-temp 72
thermals --set-target-temp off

# Switch platform thermal profile
thermals --set-profile quiet
thermals --set-profile balanced
thermals --set-profile performance

# Toggle CPU Turbo Boost
thermals --set-boost off
thermals --set-boost on

# Set battery charge threshold (50%–100%)
thermals --set-battery-limit 80

# Instant max cooling (Cooler Boost)
thermals --cooler-boost

# Run headless thermostat daemon in the background
thermals --daemon

# Test desktop push alert notification
thermals --test-alert
```

### JSON Telemetry for Status Bars

```json
{
  "system": {
    "vendor": "ASUS",
    "model": "Vivobook Go E1504FA",
    "cpu": "AMD Ryzen 5 7520U with Radeon Graphics",
    "os": "Ubuntu 26.04.1 LTS"
  },
  "temperatures": {
    "cpu": 64.4,
    "gpu": 58.0,
    "nvme": 39.9,
    "acpi": 63.0
  },
  "controls": {
    "active_profile": "balanced",
    "available_profiles": ["quiet", "balanced", "performance"],
    "target_temp_enabled": true,
    "target_temp_c": 72.0,
    "cpu_boost": true,
    "battery_charge_limit": 80
  }
}
```

---

## Systemd Service (Headless Limiter)

To run the closed-loop thermal limiter and safety monitor continuously in the background without needing the GUI:

```bash
# Enable and start user service
systemctl --user enable --now thermals.service

# Check daemon status
systemctl --user status thermals.service

# View live daemon logs
journalctl --user -u thermals.service -f
```

---

## Security & Privilege Architecture

Thermals adheres to strict privilege separation. The graphical UI and sensor monitors run entirely unprivileged as your standard desktop user:
- Reading `/sys/class/hwmon` and `/proc` requires no root permissions.
- Desktop thermal profile changes leverage standard `power-profiles-daemon` D-Bus interfaces.
- Privileged operations (direct fan PWM, battery charge threshold, CPU boost sysfs writes) are delegated to `/usr/local/sbin/asus-thermal-profile`.
- The helper strictly canonicalizes all paths with `readlink -f`, verifies paths against an exact regex whitelist (`^/sys/class/hwmon/hwmon[0-9]+/pwm[1-9]$`), validates bounded numeric parameters, and is checked with `visudo -c` during installation.

For details, review [SECURITY.md](SECURITY.md).

---

## Comparison with Existing Tools

| Capability | Thermals | TLP | auto-cpufreq | CoolerControl | asusctl |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Target Temp Limiter (Closed Loop)** | Yes | No | No | No | No |
| **Dual GUI & CLI with JSON** | Yes | CLI only | CLI only | GUI only | CLI only |
| **Non-ROG / Budget ASUS Laptop Support** | Yes | Generic | Generic | Partial | No (ROG/TUF only) |
| **Battery Charge Threshold Control** | Yes | Vendor | No | No | ROG only |
| **Zero Heavy Dependencies (Pure Python)** | Yes | No (Perl) | No | No (Rust/C++) | No (Rust) |
| **Fan Safety Floor & Thermal Override** | Yes | N/A | N/A | Curves | Profile-bound |

---

## Uninstallation

```bash
# Clean uninstall of application, service, and desktop shortcuts
bash install.sh --uninstall
```

---

## Technical Audit, Known Issues & Roadmap

Now the source files should be fetchable.

I read the privileged helper (`asus-thermal-profile`) in full and the first 1,000 lines of `app.py`. GitHub's viewer cuts off there, and the raw file is blocked to me, so I haven't seen the remaining ~1,376 lines. That's where the GUI, daemon loop, limiter logic and CLI live. Everything below is from code I actually read.

### Corrections to my earlier review

I guessed based on the README, and several guesses were wrong. The code is better than the README suggests:

- The fan safety floor, emergency override and restore-on-exit **exist** (helper and `FanSafetyController`).
- Battery charge limit **exists**.
- Throttle detection and top-process lookup **exist**.
- The limiter has min-dwell and confirm-sample logic, and hysteresis is configurable.
- The config directory has already been renamed to `~/.config/thermals`, with migration.
- The helper's input validation is solid for the commands I could see.

The README is stale against the code. It documents the old config path and the old name, shows a fixed 5°C hysteresis, and lists 9 config keys when the code has about 16.

### Bugs I can point to

**1. `grep -qw` false positives in the helper (profile command).**
`grep -qw -- "$ARG" "$PROFILE_STD_CHOICES"` treats `-` as a word boundary, so `balanced` matches inside `balanced-performance`. On a system that lists `balanced-performance` but not `balanced`, validation passes, the sysfs write fails with EINVAL, and `set -e` kills the script with an ugly error. Fix:
```bash
tr -s ' ' '\n' < "$PROFILE_STD_CHOICES" | grep -qxF -- "$ARG"
```

**2. Wrong PWM files collected.**
`hw.glob("pwm[0-9]*")` and filtering only on `_enable` also matches `pwm1_freq`, `pwm1_mode` and `pwm1_auto_point*`. Those land in `fan_pwm_files`. The helper's regex will reject them, but they pollute the UI and the restore loop. Use `re.fullmatch(r"pwm\d+", name)`.

**3. `set_cpu_epp` doesn't set EPP when power-profiles-daemon exists.**
It maps EPP to a whole PPD profile (`balance_power` collapses to `power-saver`) and returns success. The user asks for an EPP change and gets a platform-profile switch. That also fights your target limiter, which switches profiles itself. Write EPP to sysfs directly and don't route it through PPD.

**4. Dead validation in the helper's `fan` command.**
`CANON_PATH` from `readlink -f` is only checked for non-empty and never used. The regex does the real work, which is fine, but delete the pretense or enforce that the canonical path stays under `/sys/devices`.

**5. Cooler Boost state ownership is broken by design.**
`restore_all_fans` returns immediately if `_manual_active` is False. The helper's `cooler-boost` forces every `pwm[1-9]` on every hwmon to manual 255 with no record kept. There are two possible outcomes, depending on code I couldn't see. If the one-shot CLI `thermals --cooler-boost` sets `_manual_active`, its `atexit` handler restores auto the moment the process exits, so Cooler Boost lasts about zero seconds. If it doesn't set the flag, nothing ever restores the fans. Either way, manual fan state needs a persistent owner: a daemon plus a state file (`daemon_state.json` is already there), restored on startup and on service stop. In both cases a SIGKILL or crash leaves fans pinned.

**6. `pwm_enable = 2` isn't universally "auto".**
For some drivers (nct6775 variants, for instance) automatic modes use other values. Keep a per-driver map or fall back to saving and restoring the original `pwm_enable` value you read before taking control. Restoring what you read is the more robust approach.

**7. Chip classification by substring.**
`"xe" in chip`, `"core" in label`, `"gt"`, `"phy"`, `"hp"` are all fragile. The CPU check runs first and matches `"core"` in labels, so a GPU sensor labelled "GPU core" is classified as CPU. Classify by driver (`hwmon/*/device/driver` symlink) or PCI class instead of name fragments.

**8. Temperature unit heuristic.**
`v / 1000 if v > 1000 else v` is wrong by spec: `temp*_input` is always millidegrees. A real 0.9°C reading would display as 900°C, and negative values pass through. Always divide by 1000.

**9. Import-time side effects.**
`FanSafetyController.register_exit_handlers()` runs at module scope, installing SIGINT/SIGTERM/SIGHUP handlers and an `atexit` hook whenever the module is imported. That includes tests. Move it into `main()`. Also, the signal handler calls `get_registry()`, which can trigger a full hardware scan inside a signal handler if the registry is still None.

**10. Config handling is unvalidated and non-atomic.**
`load_config` copies values in without type or range checks, so `"target_temp_c": "hot"` goes straight to the control loop. `save_config` uses `write_text`, which can leave a truncated file on a crash. Clamp and validate on load, and write to a temp file then `os.replace`.

### Design and security

- **PPD plus direct sysfs writes conflict.** `set_profile` tries direct sysfs, then `powerprofilesctl`, then the helper, then pkexec. Mixing them means PPD's idea of the active profile diverges from the kernel's. Pick one authority per system and detect which.
- **`pkexec /usr/bin/tee <path>`** trains users to authenticate a generic "run tee as root" dialog. Ship a proper polkit action for your helper with a readable message and `auth_admin_keep`.
- **Throttle detection is a heuristic.** It triggers on 82°C or higher, frequency under 50% of `cpuinfo_max_freq` on policy0 only, and load of 60% or higher. A power-saving EPP can produce the same frequency drop without throttling. On Intel, read `thermal_throttle/*_throttle_count` for ground truth. For AMD, keep the heuristic but label it "possible."
- **The emergency override is a fixed 80°C.** Many Ryzen U-series chips sit above 80°C under ordinary load, so this fires constantly for them. It should be configurable, and I can't tell whether the slider or daemon reapplies manual mode right after the override fires. Check that, because it would cause oscillation at the threshold.
- **Helper scope.** The commands I read look well validated. I haven't seen `install-helper.sh`, so I can't tell who the sudoers rule covers (a specific user or an entire group), whether the file is `0440 root:root`, or whether it's checked with `visudo -c`. Verify that.
- **Architecture.** A 2,376-line single file with global mutable state (`_REGISTRY`, class-level `_manual_active`), classes that are all classmethods, and hard-coded module-level paths. This makes testing hard. Split into a package (`sensors`, `control`, `daemon`, `cli`, `gui`) and inject a `SYSFS_ROOT` so you can point it at a fake tree.
- **Possible UI blocking.** `get_available_profiles` and `get_current_profile` run `powerprofilesctl` subprocesses with 2-second timeouts. If those are called from the Tk poll loop, the UI freezes whenever PPD is slow. I couldn't see where they're called, so check it.

### Features worth building next

1. A systemd service that owns manual fan state, the limiter, and automation, with `ExecStopPost=… restore-fans`. This fixes the biggest design hole above.
2. Fan curves (temperature to PWM points) on top of the existing safety rails.
3. Intel throttle counters and AMD `amd_pstate` status for real throttle reporting.
4. Per-sensor thresholds from `crit` and `max` everywhere, not fixed 65/75/85.
5. AC vs battery profile switching.
6. CSV export of history. The tray icon and Waybar examples are easy wins given the JSON output.
7. A fake-sysfs test suite using fixtures contributed from real machines, plus an issue template that asks for `thermals --list-hardware` output.

### What I'd do first

1. Fix the `grep -qw` bug, the PWM glob, and the EPP-via-PPD behavior. They're small and real.
2. Resolve fan state ownership (daemon plus state file plus systemd stop hook).
3. Update the README to match the code, and fix the clone URL and install instructions. I flagged these last time and they still hold.
4. Move the signal and atexit setup out of import time, then add tests against a fake sysfs.
5. Read `install-helper.sh` for the sudoers scope.

To review the rest of the app (GUI thread usage, limiter loop, daemon), paste lines 1000 onward in chunks, or just the limiter/daemon and the `--cooler-boost` code path. The limiter loop is the part I'd most like to see.

---

## License

This project is licensed under the [MIT License](LICENSE).
