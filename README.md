# scanlogd for Enterprise Linux 8

Unofficial RPM packaging of [Openwall scanlogd](https://www.openwall.com/scanlogd/) 2.2.8 for **Enterprise Linux 8** with a native **systemd** service.

This repository targets **AlmaLinux 8** and is intended for RHEL 8-compatible distributions such as Rocky Linux 8. Compatibility with each distribution must be verified separately.

> **What scanlogd does:** It detects and logs certain TCP port scans. It does not block traffic by itself. The optional Fail2Ban configuration below can convert scanlogd events into iptables bans. This project is not a replacement for an IDS/IPS, a firewall, or patch management.

## Changes from upstream

The upstream scanlogd RPM specification uses a SysV init script and `chkconfig`. This packaging adapts it to EL8:

- Install a native `scanlogd.service` instead of the SysV init script.
- Use RPM's systemd scriptlet macros for installation, removal, and upgrades.
- Keep the upstream Linux TCP raw-socket implementation and scan detection logic.
- Keep scanlogd's internal privilege drop to a dedicated `scanlogd` user.
- If the packaged SPEC uses `/var/lib/scanlogd/empty` as its chroot directory, create that root-owned directory and modify the upstream compile-time path consistently.
- Install the binary, manual page, upstream README, and upstream license.

The daemon itself is from Openwall; this repository is an independent packaging effort.

## Repository layout

Suggested layout:

```text
scanlogd-el8/
├── README.md
├── SPECS/
│   └── scanlogd.spec
└── SOURCES/
    └── scanlogd.service
```

If this repository instead stores `scanlogd.spec` and `scanlogd.service` at the top level, adjust the paths in the commands below. The upstream source tarball is obtained separately and does not need to be committed to Git.

## Build with Mock

The following example works from a Linux build host with Mock installed (including a suitable EL9 build host), provided it has an AlmaLinux 8 Mock configuration. The commands assume a shell with permission to run Mock.

### 1. Install the build tools

On an AlmaLinux build host:

```bash
dnf install -y epel-release
dnf install -y mock mock-core-configs rpm-build curl git
```

Check the available configuration:

```bash
ls -l /etc/mock/alma+epel-8-x86_64.cfg
```

This guide uses `alma+epel-8-x86_64`. If only `almalinux-8-x86_64` is installed, use that configuration instead. The configuration name may vary with the installed Mock version.

### 2. Clone this repository

```bash
git clone https://github.com/redadmin-k/scanlogd-el8.git
cd scanlogd-el8
```

### 3. Retrieve the upstream source

The SPEC must reference the same source archive and service unit that are supplied to Mock. For a SPEC containing `Source0: ...scanlogd-%{version}.tar.gz` and `Source1: scanlogd.service`:

```bash
mkdir -p build/SOURCES build/SRPMS build/RPMS
curl -fL https://download.openwall.net/pub/projects/scanlogd/scanlogd-2.2.8.tar.gz -o build/SOURCES/scanlogd-2.2.8.tar.gz
cp SOURCES/scanlogd.service build/SOURCES/scanlogd.service
```

Openwall also publishes the detached source signature at:

<https://download.openwall.net/pub/projects/scanlogd/scanlogd-2.2.8.tar.gz.sign>

Verify the archive with a **trusted Openwall signing key** before production packaging. Do not assume that a successful download alone establishes authenticity.

### 4. Build the EL8 source RPM

Use Mock for the SRPM stage so that the `%{dist}` value is resolved for EL8, even when building on an EL9 host:

```bash
mock -r alma+epel-8-x86_64 --resultdir "$PWD/build/SRPMS" --buildsrpm --spec "$PWD/SPECS/scanlogd.spec" --sources "$PWD/build/SOURCES"
```

For `Version: 2.2.8` and `Release: 1%{?dist}`, the expected file is:

```text
build/SRPMS/scanlogd-2.2.8-1.el8.src.rpm
```

Always check the actual filename if the release field differs.

### 5. Build the EL8 binary RPM

```bash
mock -r alma+epel-8-x86_64 --resultdir "$PWD/build/RPMS" --rebuild "$PWD/build/SRPMS/scanlogd-2.2.8-1.el8.src.rpm"
```

Inspect the result and metadata:

```bash
ls -lh build/RPMS/
rpm -qpi build/RPMS/scanlogd-*.x86_64.rpm
rpm -qpl build/RPMS/scanlogd-*.x86_64.rpm
```

If the build fails, inspect `build/RPMS/build.log` and `build/RPMS/root.log` and check the build dependencies in the SPEC. Successful creation of an SRPM is not proof that the binary builds or runs correctly.

## Install and run on an EL8 test host

On a test AlmaLinux 8 system, copy the built RPM and install it:

```bash
dnf install -y ./scanlogd-2.2.8-1.el8.x86_64.rpm
systemctl enable --now scanlogd
systemctl status scanlogd --no-pager -l
pgrep -a scanlogd
systemctl cat scanlogd
```

Update the filename to the actual RPM produced by Mock.

scanlogd normally forks into the background, so the systemd unit typically uses `Type=forking`. It requires the ability to create a TCP raw socket before it changes root and drops privileges internally. Do not add `User=scanlogd` to the unit without carefully validating that startup sequence.

For logging and startup diagnostics:

```bash
journalctl -u scanlogd -b --no-pager
journalctl -t scanlogd -f
grep 'scanlogd\[' /var/log/messages | tail -20
```

The last command requires rsyslog to be configured to write scanlogd messages to `/var/log/messages`.

## Reproduce external TCP port-scan detection

Only scan servers you own or are explicitly authorized to test. Use a staging host and a scanner with a different public source IP. Avoid scanning from the source of your own management connection if Fail2Ban automatic banning is enabled.

From an **external test host** with Nmap:

```bash
sudo nmap -n -Pn -sS -T4 --max-retries 0 -p 22,25,53,80,110,143,443,465,587,993,995 <your-test-hostname-or-ip>
```

On the **EL8 destination host**:

```bash
journalctl -t scanlogd -f
```

Example of the expected log format (illustrative addresses, **not** an EL8 test result):

```text
scanlogd[12345]: 203.0.113.35:39864 to 192.0.2.10 ports 110, 143, 80, 22, 25, 53, ...
```

A scan is logged only when scanlogd's destination-port count, weighting, and timing conditions are met. The upstream defaults require at least seven distinct destination ports and a weighted threshold. A firewall can prevent packets from reaching scanlogd's raw socket, and LXC containers can observe only traffic delivered to their own network namespace. Not every Nmap run will be detected.

## Optional: Fail2Ban + iptables

Fail2Ban is **optional**, and its configuration is **not automatically installed** by the scanlogd RPM unless the package explicitly includes it.

The following example monitors rsyslog's `/var/log/messages`. First, install a compatible Fail2Ban package (typically available through EPEL on AlmaLinux 8) and ensure the log contains actual scanlogd events.

Create `/etc/fail2ban/filter.d/scanlogd.conf`:

```ini
[Definition]
failregex = ^.*scanlogd\[\d+\]:\s+<HOST>:\d+\s+to\s+\S+\s+ports\b
ignoreregex =
```

Create `/etc/fail2ban/jail.d/scanlogd.local`:

```ini
[scanlogd]
enabled = true
filter = scanlogd
backend = auto
logpath = /var/log/messages
maxretry = 1
findtime = 60
bantime = 600
banaction = iptables-allports
```

Check that your installed Fail2Ban provides the `iptables-allports` action; the precise action or chain name can differ by package version and local configuration.

**Before enabling bans:** configure Fail2Ban `ignoreip` for necessary management and monitoring sources and assess the spoofing risk described below. The example values are intended for controlled testing, not as a universal production policy.

Validate the filter and restart Fail2Ban:

```bash
fail2ban-regex /var/log/messages /etc/fail2ban/filter.d/scanlogd.conf --print-all-matched
fail2ban-client -t
systemctl restart fail2ban
fail2ban-client status scanlogd
```

Repeat an authorized external scan, and check both the Fail2Ban decision and actual iptables enforcement:

```bash
fail2ban-client status scanlogd
iptables -S INPUT
iptables -S | grep f2b-scanlogd
iptables -nvL f2b-scanlogd --line-numbers
```

A ban shown by Fail2Ban alone does not prove actual packet drops. Check that the firewall jump is effective and that the offending address has a DROP/REJECT rule; an increasing packet counter is evidence that traffic hit that rule. Chain names and their placement vary with the action configuration.

### Important security limitations

- **Source address spoofing:** scanlogd does not require a completed TCP connection. Spoofed packets can cause automatic blocking of an innocent address. Avoid permanent bans based only on scanlogd logs.
- **Detection blind spots:** low-rate and distributed scans can evade the thresholds; traffic dropped by upstream firewalls or local iptables INPUT rules may be invisible.
- **Ban scope:** an `iptables-allports` action can affect legitimate traffic as well as scanners; use short ban times and explicit exceptions where appropriate.
- **Defense in depth:** scanlogd does not replace a default-deny firewall, security updates, strong authentication, a WAF, or upstream DDoS protection.
- **Performance:** evaluate logging volume, the number of firewall rules, memory and CPU use, and false positives before enabling this on high-throughput services.

## Verification status

As of **2026-10-11**, **EL8-specific Mock build and runtime tests have not been reported for this repository**. Do not treat the successful AlmaLinux 9 test as EL8 certification.

| Item | EL8 status |
| --- | --- |
| Mock SRPM build | Not yet documented |
| Mock binary RPM build | Not yet documented |
| Native systemd startup | Not yet documented |
| SELinux Enforcing | Not yet documented |
| External scan detection | Not yet documented |
| Fail2Ban iptables blocking | Not yet documented |
| LXC operation | Not yet documented |

A separate EL9 experiment confirmed scanlogd logging, Fail2Ban banning, and iptables DROP counters; these results are **not** EL8 results. Update this table only after tests are actually performed on EL8.

## Troubleshooting

- **Missing `scanlogd.service` during SRPM creation:** place the service unit in the directory supplied via `--sources` and ensure it matches `Source1` in the SPEC.
- **Incorrect `.el9` or other distribution tag:** build the SRPM with `mock --buildsrpm` using the EL8 configuration, not with plain `rpmbuild -bs` on a different host.
- **The service starts but logs no scans:** check the socket and capabilities, whether SYN packets reach the destination, syslog routing, iptables INPUT policy, and scan thresholds.
- **A scan is logged but no ban occurs:** confirm the actual log path, test the regex, check `ignoreip`, and inspect `/var/log/fail2ban.log`.
- **Fail2Ban reports a ban but traffic is not dropped:** inspect firewall action rules, INPUT chain ordering, protocol family, and packet counters.
- **SELinux denials:** inspect AVC logs and fix the policy or labeling issue rather than disabling SELinux without analysis.

## Upstream and license

- Project: <https://www.openwall.com/scanlogd/>
- Source repository: <https://github.com/openwall/scanlogd>
- Original distribution files: <https://download.openwall.net/pub/projects/scanlogd/>
- License: <https://github.com/openwall/scanlogd/blob/main/LICENSE>

Openwall provides scanlogd under a heavily simplified BSD-like license allowing redistribution in source or binary form, with or without modification, and disclaiming warranties. The upstream RPM SPEC describes this as `BSD-compatible`; the license text is not identical to the standard `BSD-2-Clause` or `BSD-3-Clause` wording. Preserve the upstream license and attribution when redistributing the software, and verify packaging license metadata against the policy of the distribution where it will be submitted.

This repository is a community packaging project, **not an official Openwall release**.
