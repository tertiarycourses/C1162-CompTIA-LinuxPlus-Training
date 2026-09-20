<div align="center">

# 🐧 CompTIA Linux+ Training (XK0-006)

**Master Linux system administration hands-on — and walk out ready for the CompTIA Linux+ XK0-006 exam.**

[![Register](https://img.shields.io/badge/Register-Course-C8102E?style=for-the-badge)](https://www.tertiarycourses.com.sg/comptia-linux-exam-prep.html)
[![Course Code](https://img.shields.io/badge/Course%20Code-C1162-1F6FEB)](https://www.tertiarycourses.com.sg/comptia-linux-exam-prep.html)
[![Exam](https://img.shields.io/badge/Exam-XK0--006%20V8-red)](https://www.comptia.org/certifications/linux)
[![Duration](https://img.shields.io/badge/Duration-5%20Days%20%C2%B7%2037.5%20Hours-10B981)](#course-details)
[![Labs](https://img.shields.io/badge/Hands--On%20Labs-30-7C3AED)](#course-topics--labs)

**[📝 Register for this course →](https://www.tertiarycourses.com.sg/comptia-linux-exam-prep.html)**

</div>

---

## About This Course

This 5-day course teaches you to configure, manage, secure, automate and troubleshoot Linux servers, mapped 1:1 to the five **CompTIA Linux+ XK0-006 (V8)** exam domains. It is built for system administrators, DevOps engineers, cloud engineers and IT professionals who want job-ready Linux skills plus a clear path to the Linux+ certification.

Every one of the **30 step-by-step labs** runs free in the browser on the [Killercoda Ubuntu Playground](https://killercoda.com/playgrounds/scenario/ubuntu) — no local install, no virtual machine, no credit card required.

## Learning Outcomes

| # | Learning Outcome |
|---|---|
| LO1 | Explain core Linux concepts — boot process, Filesystem Hierarchy Standard, device management and virtualization — and manage storage, networking and shell operations. |
| LO2 | Manage files, local user and group accounts, processes and jobs, software packages, systemd services and containerized applications on a Linux server. |
| LO3 | Apply Linux security: authentication/authorization (sudo, PAM), firewalls, OS and account hardening, cryptography and compliance/audit procedures. |
| LO4 | Automate administration with Ansible, Bash and Python, apply Git version control, and use AI assistants responsibly and securely. |
| LO5 | Monitor a Linux system and analyze and troubleshoot hardware, storage, network, security and performance issues using the right diagnostic tools. |

## Course Topics & Labs

The course follows the XK0-006 exam blueprint. Each lab has its own folder under [labs/](labs/) with a full step-by-step `README.md`.

| Exam Domain | Weight | Hands-On Labs |
|---|---|---|
| **1 · System Management** | 23% | [Lab 1 — Boot & FHS](labs/lab-01-boot-fhs/) · [Lab 2 — Kernel & Devices](labs/lab-02-kernel-devices/) · [Lab 3 — Storage & LVM](labs/lab-03-storage-lvm/) · [Lab 4 — Networking](labs/lab-04-networking/) · [Lab 5 — Shell & Text](labs/lab-05-shell/) · [Lab 6 — Backup & Restore](labs/lab-06-backup-restore/) · [Lab 7 — Virtualization](labs/lab-07-virtualization/) |
| **2 · Services and User Management** | 20% | [Lab 8 — Files & Directories](labs/lab-08-files-directories/) · [Lab 9 — Accounts & Groups](labs/lab-09-users-groups/) · [Lab 10 — Processes & Jobs](labs/lab-10-processes-jobs/) · [Lab 11 — Packages](labs/lab-11-packages/) · [Lab 12 — systemd](labs/lab-12-systemd/) · [Lab 13 — Containers](labs/lab-13-containers/) |
| **3 · Security** | 18% | [Lab 14 — AAA: sudo, PAM, Polkit](labs/lab-14-aaa-sudo-pam/) · [Lab 15 — Firewalls](labs/lab-15-firewall/) · [Lab 16 — OS Hardening](labs/lab-16-hardening/) · [Lab 17 — Account Hardening](labs/lab-17-account-hardening/) · [Lab 18 — Cryptography](labs/lab-18-crypto/) · [Lab 19 — Compliance & Audit](labs/lab-19-compliance-audit/) |
| **4 · Automation, Orchestration & Scripting** | 17% | [Lab 20 — Ansible IaC](labs/lab-20-ansible/) · [Lab 21 — Bash Scripting](labs/lab-21-bash-scripting/) · [Lab 22 — Python for Sysadmin](labs/lab-22-python/) · [Lab 23 — Git](labs/lab-23-git/) · [Lab 24 — Responsible AI Use](labs/lab-24-ai-best-practices/) |
| **5 · Troubleshooting** | 22% | [Lab 25 — Monitoring](labs/lab-25-monitoring/) · [Lab 26 — Storage/OS Triage](labs/lab-26-troubleshoot-storage/) · [Lab 27 — Network Triage](labs/lab-27-troubleshoot-network/) · [Lab 28 — Security Triage](labs/lab-28-troubleshoot-security/) · [Lab 29 — Performance Triage](labs/lab-29-troubleshoot-performance/) · [Lab 30 — Capstone](labs/lab-30-capstone/) |

**How to run a lab:** open the [Killercoda Ubuntu Playground](https://killercoda.com/playgrounds/scenario/ubuntu), pick a lab folder, and follow its `README.md` step by step. Reset the playground between labs that change kernel, firewall or systemd state.

## Tools

All tooling is **100% free** — the bulk runs inside the disposable Killercoda VM via `apt`/`dnf` or open-source binaries.

| Tool | Used In | Link |
|---|---|---|
| Killercoda Ubuntu Playground | Every lab | <https://killercoda.com/playgrounds/scenario/ubuntu> |
| Regex Generator | Lab 21 | <https://alfredang.github.io/regexgenerator/> |
| ShellCheck (online) | Lab 21 | <https://www.shellcheck.net/> |
| CompTIA XK0-006 Exam Objectives (PDF) | Every lab | in this repo |
| Practice Exam — CompTIA Linux+ | Exam prep | <https://exams.tertiaryinfotech.com/practice-exams/comptia/comptia-linux-plus> |

Full tool list with install commands: [labs/tools.md](labs/tools.md). Lab references and further practice: [labs/README.md](labs/README.md).

## Repository Structure

```
.
├── courseware/                  # Course deliverables
│   ├── PPT-CompTIA-Linux-Plus-XK0-006-v1.pptx   # 517-slide training deck (+ PDF)
│   ├── LP-CompTIA-Linux-Plus-XK0-006.docx    # 5-day Lesson Plan (+ PDF)
│   └── LG-CompTIA-Linux-Plus-XK0-006.docx    # Learner Guide (+ PDF)
├── labs/                        # 30 hands-on labs — one folder per lab
│   ├── lab-01-boot-fhs/ … lab-30-capstone/
│   ├── README.md                # lab index + references
│   └── tools.md                 # free-tool catalogue
└── CompTIA Linux+ XK0-006 V8 Exam Objectives (4.0).pdf
```

## Course Details

| | |
|---|---|
| **Course Title** | CompTIA Linux+ Training (XK0-006) |
| **Course Code** | C1162 |
| **Certification Exam** | CompTIA Linux+ XK0-006 V8 |
| **Duration** | 5 days · 37.5 hours (9:30am – 5:30pm) |
| **Mode** | Instructor-led, hands-on labs on the free Killercoda Ubuntu Playground |
| **Trainer** | Dr. Alfred Ang |
| **Training Provider** | Tertiary Infotech Academy Pte Ltd (UEN 201200696W) |

## License & Attribution

© Tertiary Infotech Academy Pte Ltd. Course materials are provided for enrolled learners.
CompTIA and Linux+ are trademarks of CompTIA, Inc.

---

<div align="center">

**[📝 Register for the CompTIA Linux+ Training →](https://www.tertiarycourses.com.sg/comptia-linux-exam-prep.html)**

</div>
