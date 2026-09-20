# Lab 11 — Software & Package Management

This lab covers CompTIA Linux+ XK0-006 Objective 2.4 (Given a scenario, manage software and packages). You will use `apt` and `dpkg`, add a third-party repository with GPG verification, demo `dnf`/`rpm` inside a container, install language-specific packages, and explore sandboxed package formats.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — apt basics

```bash
apt update
apt install -y tree jq
apt-cache policy jq
dpkg -l | grep -E "tree|jq"
dpkg -L tree | head
```

`apt update` refreshes package indexes, `apt install` resolves dependencies, and `apt-cache policy` shows installed and candidate versions. `dpkg -l` lists installed packages; `dpkg -L` lists files owned by a package.

---

## Step 2 — Add a third-party repo with GPG key

```bash
apt install -y curl gnupg lsb-release
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://apt.releases.hashicorp.com/gpg \
  | gpg --dearmor -o /etc/apt/keyrings/hashicorp.gpg
echo "deb [signed-by=/etc/apt/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  > /etc/apt/sources.list.d/hashicorp.list
apt update
apt-cache policy terraform | head
```

Modern Debian/Ubuntu uses signed-by per-repo keyrings instead of the deprecated `apt-key`. The GPG signature guarantees package authenticity.

---

## Step 3 — Remove packages

```bash
apt remove -y tree
apt purge -y jq
apt autoremove -y
```

`remove` keeps configuration files; `purge` deletes them too; `autoremove` cleans up orphaned dependencies.

---

## Step 4 — RPM world inside a container

```bash
apt install -y docker.io
systemctl start docker
docker run --rm rockylinux:9 bash -c "
  dnf -y install nano > /dev/null &&
  rpm -q nano &&
  rpm -V nano || true &&
  rpm -qa | head
"
```

`dnf` is the Red Hat/Fedora package manager. `rpm -q` queries installation status and `rpm -V` verifies file integrity against the package database.

---

## Step 5 — Language-specific package managers

```bash
apt install -y python3-pip nodejs npm cargo
pip3 install --user --break-system-packages requests
python3 -c "import requests; print(requests.__version__)"
npm install -g cowsay
cowsay "Linux+"
cargo --version
```

`pip` installs Python libraries (user-scope by default), `npm -g` installs Node.js CLI tools globally, and `cargo` is the Rust toolchain's build and package manager.

---

## Step 6 — update-alternatives

```bash
update-alternatives --display editor
update-alternatives --list editor
```

`update-alternatives` manages symlinks for commands that have multiple providers (editor, java, python, etc.). `--display` shows current and candidate links.

---

## Step 7 — Sandboxed package formats

```bash
which snap && snap list 2>/dev/null || echo "snap not preinstalled on this image"
apt install -y flatpak
flatpak --version
flatpak remotes
```

Snap (Canonical) and Flatpak (freedesktop) ship apps with bundled dependencies in confined sandboxes — useful for desktop apps and isolating runtime versions.

---

## Step 8 — Cleanup

```bash
rm -f /etc/apt/sources.list.d/hashicorp.list /etc/apt/keyrings/hashicorp.gpg
apt update
npm uninstall -g cowsay
pip3 uninstall -y --break-system-packages requests
apt remove -y flatpak nodejs npm cargo python3-pip docker.io
apt autoremove -y
```

The third-party repo, language packages, and Docker are all removed to leave a clean playground.

---

## What you learned
- Using `apt`, `apt-cache`, and `dpkg` to manage Debian packages
- Securely adding third-party repositories with signed-by GPG keyrings
- Running `dnf`/`rpm` workflows in a container and using language and sandbox package managers

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- APT / dpkg — https://wiki.debian.org/Apt
- HashiCorp APT repository — https://apt.releases.hashicorp.com/
- Rocky Linux container image — https://hub.docker.com/_/rockylinux
- Flatpak — https://flatpak.org/
