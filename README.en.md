<!-- synced-with: README.md @ 6c41533 (2026-09-12) -->

[![README en español](https://img.shields.io/badge/README-Espa%C3%B1ol-lightgrey)](README.md)
[![README in English](https://img.shields.io/badge/README-English-blue)](README.en.md)

# Cat-ServerFullReport

**As-built inventory and health check for Linux servers, in a single bash script.**

[![Bash 4.0+](https://img.shields.io/badge/Bash-4.0%2B-4EAA25?logo=gnubash&logoColor=white)](#requirements)
[![Linux](https://img.shields.io/badge/Linux-Debian%20%7C%20Ubuntu%20%7C%20RHEL%20%7C%20SUSE-FCC624?logo=linux&logoColor=black)](#compatibility)
[![MIT License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Read-only](https://img.shields.io/badge/Read--only-does%20not%20modify%20the%20host-brightgreen.svg)](#security)
[![No dependencies](https://img.shields.io/badge/Dependencies-none-lightgrey.svg)](#requirements)
[![Output](https://img.shields.io/badge/Output-HTML%20%7C%20JSON%20%7C%20Markdown-blue.svg)](#output-formats)

Collects **41 configuration sections** from a Linux server, analyses them with **14 rule topics** mapped to CIS and ISO 27001 controls, and produces a navigable HTML report with an executive summary of findings at the top. It works both as documentation (a formal as-built, an audit annex) and as a review (what is wrong with this server today).

One file. No modules, no installation, no internet access, no `jq` or Python. Copy and run.

> **Note on language:** the generated report, the script's console messages and the rest of the documentation are currently in **Spanish**. An English report is on the roadmap. Section IDs, option names and column names are language-neutral, so the report is usable with this README at hand — but if you need English output, that is not available yet.

<p align="center">
  <img src="docs/img/reporte-light.png" alt="Report in light theme: health score, KPI cards and executive summary of findings" width="100%">
</p>

<details>
<summary>See it in dark theme</summary>
<p align="center">
  <img src="docs/img/reporte-dark.png" alt="The same report in dark theme" width="100%">
</p>
</details>

> **Status: `v0.1.0-slice` — a working proof of concept, not a finished MVP.**
> The script runs end to end and produces complete reports against real machines,
> but it has only been validated on Ubuntu, CentOS Stream 9, openSUSE Leap and Debian 12. See [Compatibility](#compatibility).

It is the Linux sibling of [Get-ServerFullReport](https://github.com/KikeMuller/Get-ServerFullReport) (Windows / PowerShell): the same report schema, the same visual language in the HTML, and the same discipline of never asserting anything about data that could not be read.

---

## Contents

- [Why](#why)
- [Why bash and not Python](#why-bash-and-not-python)
- [Installation](#installation)
- [Usage](#usage)
- [Options](#options)
- [Running it as root](#running-it-as-root)
- [Auditing a remote server](#auditing-a-remote-server)
- [What it collects](#what-it-collects)
- [The findings engine](#the-findings-engine)
- [Output formats](#output-formats)
- [Shareable version (`--redact`)](#shareable-version---redact)
- [Report screenshots](#report-screenshots)
- [Requirements](#requirements)
- [Compatibility](#compatibility)
- [Security](#security)
- [Known limitations](#known-limitations)
- [Contributing](#contributing)
- [License](#license)

---

## Why

On Linux the information exists, but it is scattered across twenty different commands: `lsblk`, `ss`, `systemctl`, `sysctl`, `dmidecode`, `journalctl`, `sshd -T`, `auditctl`. Gathering it by hand takes half an hour per server, and the result is a pile of console output nobody is going to read twice.

`cfg2html` solves the collection part, but it delivers a dump: it does not evaluate anything, does not prioritize, does not say what is wrong. Commercial inventory tools cost money and require deploying an agent.

This script fills that gap: one file you copy to the server, run, and end up with a complete document **and** a prioritized list of what needs fixing, each one with the corresponding CIS or ISO control reference.

## Why bash and not Python

Because there are servers that do not have Python installed, sometimes by security policy. A dependency-free bash script runs on any Linux server exactly as it ships from the factory; a Python one forces you to check the interpreter, version and modules before you can audit anything — right on the machines where you least want to install things.

That is where the three rules that define the project come from:

- **A single file.** Copy it and run it. No package, no installer, no `pip install`.
- **Zero dependencies outside the base system.** Only coreutils, procps, systemd and utilities that are already present on any server distro. In particular, **it does not use `jq`**: the internal format is TSV, not JSON, precisely so it doesn't depend on it.
- **Read-only.** The script never modifies the audited machine. It does not start services, does not open ports, does not write outside its own temporary files.

## Installation

There is no installation. Download the file and make it executable:

```bash
curl -fsSLO https://raw.githubusercontent.com/KikeMuller/Cat-ServerFullReport/main/cat-serverfullreport
chmod +x cat-serverfullreport
```

Or clone the repository, or copy the file over with `scp` from wherever. Any of the three works: it is a single text file with nothing else around it.

## Usage

```bash
# Full inventory of the local machine
sudo ./cat-serverfullreport -o /tmp

# Quick security sweep: accounts and firewall only
sudo ./cat-serverfullreport --sections 1.8,1.14

# Without the package inventory or the Python section, the long-running ones
sudo ./cat-serverfullreport --skip-sections 1.6,1.19

# All three output formats
sudo ./cat-serverfullreport --format HTML,JSON,MD -o /tmp

# Version for sharing outside the organization
sudo ./cat-serverfullreport --redact -o /tmp

# Everything, including the slow parts
sudo ./cat-serverfullreport --include-missing-updates --scan-git-repos --event-log-days 14
```

Produces `AsBuilt_<hostname>_<date>.html` in the folder you specify. It opens in any browser; the HTML is self-contained (CSS and JS embedded), so it can be attached to an email or saved with nothing else around it.

With `-q` the only output is the generated paths, one per line, which lets you chain it:

```bash
path="$(sudo ./cat-serverfullreport -q -o /tmp)"
```

## Options

| Option | What it does |
|---|---|
| `-o`, `--output DIR` | Folder to write the report to. Defaults to the current directory. |
| `-q`, `--quiet` | Silences progress messages. Still prints the path of each generated file to stdout, so it can be captured. |
| `--format LIST` | Comma-separated output formats: `HTML`, `JSON`, `MD`. Defaults to `HTML`. |
| `--redact` | Shareable version: masks IPs, accounts, home paths and serial numbers. |
| `--sections LIST` | Only these sections, by id prefix, comma-separated. |
| `--skip-sections LIST` | Skips these sections. |
| `--event-log-days N` | Days to look back for journal errors. Defaults to 7. |
| `--scan-git-repos` | Looks for Git repositories under `/home` and `/root`. Off by default because it can be slow. |
| `--include-missing-updates` | Queries pending updates. Off by default because it can be slow. |
| `-h`, `--help` | Help. |

## Running it as root

**Running it as root or with `sudo` is recommended.** The script works without privileges and never fails for lack of them, but several sections come back incomplete because the kernel simply does not hand that data to a regular user:

| Section | Without root | With root |
|---|---|---|
| 1.7 SSH server | Only the config file | Full **effective** configuration (`sshd -T`) |
| 1.14 Firewall | Only whether the service is active | The loaded rules |
| 1.16 Hardware | Everything except the serial number | Includes the serial number (`dmidecode`) |
| 1.8.4 Audit policy | Empty | `auditd` rules |
| 1.10.1 Scheduled tasks | Only the current user's crontab | Crontabs for all users |
| 1.4.3 Listening ports | Ports, no owner | Ports with the process that opened them |
| 1.9 SELinux / AppArmor | General status | Loaded profiles |
| 1.17.4 SMART health | Not available | Status of each physical disk |

Without root the report is still generated, and each affected section explicitly states why it is incomplete. It never makes up a value or marks something as correct that could not be verified.

## Auditing a remote server

The script **always runs locally, on the machine being audited**. For a remote server, copy it there and run it there:

```bash
scp cat-serverfullreport server:/tmp/
ssh server 'chmod +x /tmp/cat-serverfullreport && sudo /tmp/cat-serverfullreport -o /tmp'
scp server:/tmp/AsBuilt_*.html ./
```

This is not a workaround: a single dependency-free file copies anywhere and runs. A native remote mode over multiplexed SSH remains an open idea, with no date attached (see [CONTRIBUTING.md](CONTRIBUTING.md)).

## What it collects

**41 sections** grouped by topic. The full detail, with the exact column names for each one, is in **[docs/INVENTARIO_SECCIONES.md](docs/INVENTARIO_SECCIONES.md)**.

| Group | Sections |
|---|---|
| **System** | Operating system and kernel · systemd services · Installed packages · Pending updates · Pending reboot |
| **Storage** | Disk usage · Disks and partitions · LUKS encryption |
| **Network** | IPv4 interfaces · Routing table · Listening ports |
| **Security** | UID 0 accounts · sudo groups · Password policy · Local users · Audit policy · SUID binaries · Capabilities · SELinux / AppArmor · Firewall · Kernel hardening (sysctl) |
| **TLS** | Installed certificates · OpenSSL TLS configuration |
| **Scheduling** | Cron jobs · systemd timers |
| **Health** | Journal errors · Failed logins · Top 15 processes by memory · SMART health · Unexpected shutdowns |
| **Platform** | Host hardware · Server roles (nginx / Apache / Samba / NFS) · NTP synchronization · Python and interpreters · pip packages · Virtual environments · Git repositories · Docker containers |
| **Protection** | Antivirus / EDR detected · Backup agents detected |

When a role is not installed, the section **is still recorded** with an explanation. The index stays complete, and the as-built explicitly documents that the role does not exist on the machine — which is information just as valid as its configuration.

## The findings engine

**14 evaluation topics** that produce findings classified as `CRIT`, `WARN`, `INFO` and `OK`, with category, concrete evidence, a recommendation and a reference to the corresponding control (CIS / ISO 27001).

The central principle, inherited from the Windows version:

> No rule emits a finding — negative or positive — about data it could not read. A false `OK` is worse than silence, because the administrator will trust it.

In practice, every rule first checks that the data exists and is usable; if it is not, it says nothing. A real example of why this matters: `snap`'s read-only mounts always report 100% usage by design. Evaluating them like a normal disk produced **eight false `CRIT`s** on any Ubuntu with snapd installed. They are now explicitly excluded.

The health score starts at 100 and subtracts 12 for each `CRIT` and 4 for each `WARN`. `INFO` findings do not subtract.

## Output formats

```bash
sudo ./cat-serverfullreport --format HTML,JSON,MD -o /tmp
```

The three files from the same run share the timestamp in their name, so they correspond to each other unambiguously.

| Format | What it's for |
|---|---|
| `HTML` | Navigable, self-contained report. The one you read and share. |
| `JSON` | For automated ingestion: CMDB, inventory, comparison between runs. |
| `MD` | For pasting into a ticket, a wiki or a pull request without attaching files. |

The **JSON uses the same schema as the Windows version** (`Metadata`, `Capabilities`, `Findings`, `FindingsSummary`, `Sections`), so a tool built to consume a Windows server's inventory works the same way with a Linux one. It comes out as UTF-8 without BOM and with ISO 8601 dates.

Two JSON design decisions worth knowing about:

- **Every value under `Data` and `Findings` is a string.** The only real numbers are `RowCount`, the `FindingsSummary` counters and `TiempoTotalSegundos`. Emitting bare numbers from bash risks producing invalid JSON when the value is `N/D`, `221.1M` or empty.
- **Invalid UTF-8 bytes are sanitized** before the file is closed. The JSON spec requires valid UTF-8, so a single invalid byte — Linux filenames are byte strings, not guaranteed text — would make a strict parser reject the entire report. A browser, on the other hand, tolerates those bytes, which is why the HTML does not need the sanitizing step.

## Shareable version (`--redact`)

Masks what identifies the machine and its users, while keeping what makes the audit useful:

| Masked | Example |
|---|---|
| IPv4 and IPv6 addresses | `192.0.2.58%eth0` → `192.0.x.x%eth0` |
| User accounts | `ana,pedro` → `a****,p****` |
| Home directory paths | `/home/ana/proyectos/x` → `/home/a****/...` |
| Serial numbers | `7X2K9Q1` → `7X****` |
| MAC addresses and long hashes | inside the findings text |

**What is deliberately NOT masked**, because it does not identify anyone and is the actual content of the audit:

- System paths (`/usr/bin/sudo` in the SUID binaries table would stay readable).
- Wildcard and loopback addresses (`0.0.0.0`, `127.0.0.1`): telling apart "listening on every interface" from "listening only locally" is exactly the data point that makes the ports section useful.
- Package and kernel versions. This matters more than it looks: the kernel version `6.6.87.2` has the shape of an IPv4 address, and masking it by accident would destroy a central piece of the inventory.

`--redact` applies equally to all three output formats.

## Report screenshots

The screenshots in this README come from a real run as root on an Ubuntu machine, with `--redact` on and the hostname replaced with `SRV-DEMO-01`.

### Front page: score, KPIs and executive summary

[`docs/img/reporte-light.png`](docs/img/reporte-light.png) — The first thing you see when you open the report:

- **Filterable sidebar index** with the row count for each section, so you can tell at a glance where there is data and where there isn't.
- **KPI cards**: health score, count of criticals and warnings, uptime, RAM, busiest disk and pending patches.
- **Executive summary of findings** with severity filters and a `Ver →` link from each finding to the section it came from. Every row carries concrete evidence (`PASS_MAX_DAYS=99999 en /etc/login.defs`), not a generic statement.

### Dark theme

[`docs/img/reporte-dark.png`](docs/img/reporte-dark.png) — The same report in dark theme. The preference is saved in `localStorage`, so it survives reloads and sharing the file.

### Data tables and masking

<p align="center">
  <img src="docs/img/reporte-tabla.png" alt="Table of listening ports with masked IP addresses and wildcard addresses preserved" width="100%">
</p>

[`docs/img/reporte-tabla.png`](docs/img/reporte-tabla.png) — A data section, with two things worth a closer look:

- **Sortable headers** and the row count next to each subsection's title.
- **`--redact` in action**: real addresses show up as `192.168.x.x`, including the case with a zone suffix (`192.168.x.x%wlp1s0`), while `0.0.0.0` stays intact on purpose — that is what lets you tell a service exposed on every interface apart from one that only listens locally.

This screenshot was taken with `--sections 1.4.3,1.16`, so it also shows section filtering: the sidebar index is left with only the two requested, and KPI cards that depend on sections that were not collected say `N/D` with the reason, instead of making up a zero.

> **[See a full example report](docs/reporte-ejemplo.html)** — download it and open it in a browser (GitHub does not render embedded HTML).

## Requirements

- **bash 4.0 or later.** It uses associative arrays, `${var,,}` and parameter expansion with character classes. Any server distro since 2010 meets this.
- **coreutils, procps and `util-linux`**, which are part of the base system.
- **systemd** for the services and timers sections. On a machine with a different init system, those sections are recorded with the corresponding explanation instead of failing.
- Optional, each one enables a section: `dmidecode` (serial number), `smartmontools` (SMART health), `auditd` (audit policy), `openssl` (TLS configuration), `docker` (containers).

Strict POSIX `sh` compatibility is not a goal.

## Compatibility

| Distribution | Status |
|---|---|
| Ubuntu 24.04 / 26.04 | **Tested**, including a run as root with `LANG=es_ES.UTF-8` |
| Debian 12 (bookworm) | **Tested** on a real VM (KVM). See [Tested Debian compatibility](#tested-debian-compatibility). |
| CentOS Stream 9 | **Tested** on a real VM (KVM), including the `dnf`, `firewalld` and `update-crypto-policies` branches. See [Tested RHEL compatibility](#tested-rhel-compatibility). |
| RHEL / Rocky / Alma / Fedora | Not tested directly, but they share a base with CentOS Stream 9 (same `dnf`, `systemd`, `firewalld`, SELinux) |
| openSUSE Leap 15.6 | **Tested** on a real VM (KVM), including the `zypper` branch with bash 4.4 (the project's stated minimum). See [Tested SUSE compatibility](#tested-suse-compatibility). |
| SLES / openSUSE Tumbleweed | Not tested directly, but they share `zypper`/`rpm` with openSUSE Leap |
| Alpine / Arch | **Partial.** Installed packages are listed (`apk`, `pacman`), but pending updates are not: that section reports that the manager has no query implemented |

If you try it on one of the unvalidated ones, a report on the result is very welcome.

The script forces `LC_ALL=C` on every external command, so it behaves the same on servers configured in any language. This came out of a real problem: on a machine with `LANG=es_ES.UTF-8`, `lsblk` returned sizes like `221,1M` instead of `221.1M`, and the parsers failed silently.

### Tested RHEL compatibility

The RHEL family was tested on a real VM (CentOS Stream 9, official cloud image on KVM/libvirt), not just against documentation. The test found and fixed a real bug: the firewall section never tried to read `firewalld` rules — it only tried `ufw`, `nft` and `iptables` — so on a typical RHEL server, running as root with firewalld active, it said "could not be read without root privileges", which was false: privileges were there, the code branch was missing. Fixed by adding `firewall-cmd --list-all-zones` and distinguishing the real reason (no privileges / no tool installed / tool detected but no data) instead of a fixed message.

Confirmed on that same VM:

- `dnf check-update` interpreted correctly (exit code 0 with no output = no pending updates; code 100 = updates available, per dnf's own documented behavior).
- `update-crypto-policies --show` correctly returning `DEFAULT`.
- SELinux via `getenforce` (`Enforcing`), this family's counterpart to AppArmor.
- Degradation without privileges: same behavior as on Ubuntu, no bash errors.

Rocky Linux, AlmaLinux and RHEL itself were not tested directly, but they share a binary base with CentOS Stream 9 (same `dnf`, `systemd`, `firewalld`, SELinux), so the residual risk is low.

### Tested SUSE compatibility

The SUSE family was tested on a real VM (openSUSE Leap 15.6, official cloud image on KVM/libvirt). The test found and fixed two real bugs in the `zypper` pending-updates branch:

- **No repository refresh beforehand.** Unlike the `apt` branch, which refreshes the index before querying (with the same documented rationale in the code: avoid false negatives against a stale cache), the `zypper` branch did not. On a freshly provisioned machine, the first `zypper` query needed to build one repository's cache — took several real seconds — and the query's `timeout` killed it before it printed anything: the report said "no pending updates" when there were actually four.
- **The `awk` did not exclude the header row of `zypper list-updates`'s table.** The row `S | Repository | Name | Current Version | Available Version | Arch` satisfies the same conditions as a real data row and slipped through as if it were a package named "Name".

Fixed by adding a rooted `zypper refresh` (same pattern as `apt-get update`) and excluding the header by exact field content, not by position.

Confirmed on that same VM:

- **bash 4.4.23**, the oldest version among the three distributions tested — validates the project's stated `bash 4.0+` floor in practice, not just in theory.
- `rpm -qa --queryformat` (`zypper`'s actual backend) correctly listing 591 packages.
- `iptables -L -n` (the firewall detected on this image, with no firewalld or nft installed) returning real rules.
- AppArmor via `aa-status`, a third live confirmation of the same mechanism as Ubuntu, distinct from RHEL's SELinux.

SUSE Linux Enterprise Server (SLES) and openSUSE Tumbleweed were not tested directly, but they share `zypper` and the `rpm` format with openSUSE Leap.

### Tested Debian compatibility

Debian 12 (bookworm) was tested on a real VM (KVM/libvirt). The test found and fixed a real bug that affects **every** distribution tested, not just Debian: several admin tools (`aa-status`, `dmidecode`, `ufw`, `nft`, `iptables`, `getenforce`) live in `/usr/sbin` by convention, and that directory is **not in the `PATH`** of an unprivileged user on Debian/Ubuntu (confirmed live: `/usr/local/bin:/usr/bin:/bin:/usr/games`, no trace of `sbin`). Capability detection used a bare `command -v`, so without root the script reported "No SELinux or AppArmor detected on this machine" when AppArmor was in fact installed — a false claim about data that was never actually searched for in the right place, the same class of problem as the `firewalld` bug fixed earlier, just at the detection layer instead of the message layer.

This exact patch already existed, but only for `sshd`. It was generalized into a function (`herramienta_disponible`) that also checks `/usr/sbin`, `/sbin` and `/usr/local/sbin`, and applied to all seven admin binaries the script detects.

Confirmed on that same VM:

- **A genuinely troublesome real-world install**: the official Debian cloud image turned out to have several obstacles to an automated `cloud-init` boot (GRUB hung without a video device, `cloud-init.target` wasn't wired into the boot sequence by default, and the `NoCloud` datasource never detected the seed CD-ROM). None of this is a problem with the script — it's test infrastructure, documented here because it explains why Debian was tested after CentOS and openSUSE.
- `apt`/`dpkg-query` correctly listing 324 packages, with no false negatives on pending updates (manually verified against `apt list --upgradable`).
- AppArmor via `aa-status` (11 profiles, all in enforce mode) — a fourth live confirmation of the same mechanism.
- `bash 5.2.15`, no issues.

## Security

- **Read-only, no exceptions.** It does not start services, does not open ports, does not write outside its temporary files, which it cleans up on exit.
- **It never opens network connections to third parties.** The only exception is `--include-missing-updates`, which queries the repositories already configured on the machine, and is off by default.
- **Credentials embedded in Git URLs are masked** always, even without `--redact`.
- **Every external command goes through a layer with `timeout`.** Without that, an NFS mount with the server down, or an unreachable LDAP entry in `nsswitch.conf`, would hang the entire report.
- **A report without `--redact` contains sensitive data**: local accounts, internal IPs, listening ports, the machine's serial number. Treat it as such, and use `--redact` for any copy that leaves the organization.

## Known limitations

- **It does not determine which TLS protocols the system actually accepts.** Section 1.11.2 reports the OpenSSL version and explicit configuration, but it does not issue a compliance verdict. The only reliable way to know is to open a real TLS connection, and the script is read-only. Documented in detail in the code itself.
- **Pending updates with `apk` and `pacman`** is not implemented.
- **Rocky Linux, AlmaLinux and RHEL itself were not tested directly** (only CentOS Stream 9; see [Tested RHEL compatibility](#tested-rhel-compatibility)).
- **SLES and openSUSE Tumbleweed were not tested directly** (only openSUSE Leap 15.6; see [Tested SUSE compatibility](#tested-suse-compatibility)).

Compared to the Windows version, these capabilities are still missing:

| Missing | Windows equivalent |
|---|---|
| CSV export (it already produces HTML, JSON and Markdown) | `-Format CSV` |
| Comparison against a previous run (drift) | `-BaselinePath` |
| Auditing several machines in a single run | `-ComputerName` with several names, `-ThrottleLimit` |
| Performance counter sampling | `-PerfSampleSeconds` |

## Contributing

The project's rules, the templates for adding sections and rules, and the test procedure are in **[CONTRIBUTING.md](CONTRIBUTING.md)**. Almost all the hard rules in that file come from a real bug that took time to find, and several of them fail silently: it is worth reading them before touching the script.

The change history is in [CHANGELOG.md](CHANGELOG.md).

## License

MIT. See [LICENSE](LICENSE).
