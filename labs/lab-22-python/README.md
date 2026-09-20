# Lab 22 — Python for System Administration

This lab introduces Python for Linux administration tasks as required by CompTIA Linux+ XK0-006 Objective 4.3 (Given a scenario, perform basic Python scripting tasks). You will create a virtual environment, exercise the core data types, walk the filesystem, parse JSON, call a public API with `requests`, and lint with `black` and `flake8`.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Check Python and create a virtual environment

```bash
python3 --version
apt update && apt install -y python3-venv python3-pip
python3 -m venv ~/lab-venv
source ~/lab-venv/bin/activate
which python
pip install --upgrade pip
```

A venv keeps project packages isolated from the system Python so `pip install` cannot break OS tools.

---

## Step 2 — Install dependencies

```bash
pip install requests black flake8
pip list
mkdir -p ~/pylab && cd ~/pylab
```

`requests` is the de-facto HTTP client. `black` is an opinionated formatter; `flake8` checks PEP 8 style and obvious bugs.

---

## Step 3 — Exercise core data types

```bash
cat > ~/pylab/types_demo.py <<'EOF'
"""Demonstrate the built-in Python data types."""

flag: bool = True
count: int = 42
ratio: float = 3.14
name: str = "linuxplus"
items: list = ["nginx", "ssh", "cron"]
meta: dict = {"distro": "ubuntu", "version": 22.04}

print(type(flag), flag)
print(type(count), count)
print(type(ratio), ratio)
print(type(name), name.upper())
print(type(items), items, "len=", len(items))
print(type(meta), meta["distro"])
EOF
python ~/pylab/types_demo.py
```

Type hints (`flag: bool`) are optional but help readers and tools like `mypy` catch bugs early.

---

## Step 4 — Walk /etc, count files, emit JSON

```bash
cat > ~/pylab/scan_etc.py <<'EOF'
"""Walk /etc and print a JSON summary."""
import json
import os
from pathlib import Path


def scan(root: str) -> dict:
    total_files = 0
    total_dirs = 0
    by_ext: dict[str, int] = {}
    for dirpath, dirnames, filenames in os.walk(root):
        total_dirs += len(dirnames)
        total_files += len(filenames)
        for f in filenames:
            ext = Path(f).suffix or "(none)"
            by_ext[ext] = by_ext.get(ext, 0) + 1
    return {
        "root": root,
        "files": total_files,
        "dirs": total_dirs,
        "top_ext": dict(sorted(by_ext.items(), key=lambda kv: -kv[1])[:5]),
    }


if __name__ == "__main__":
    print(json.dumps(scan("/etc"), indent=2))
EOF
python ~/pylab/scan_etc.py
```

`os.walk` yields `(dirpath, dirnames, filenames)` tuples. `json.dumps` serialises the dict for downstream tools.

---

## Step 5 — Parse a JSON file with the `json` module

```bash
cat > ~/pylab/sample.json <<'EOF'
{
  "users": [
    {"name": "alice", "role": "admin"},
    {"name": "bob",   "role": "dev"}
  ]
}
EOF

cat > ~/pylab/read_json.py <<'EOF'
import json
from pathlib import Path

data = json.loads(Path("/root/pylab/sample.json").read_text())
for u in data["users"]:
    print(f"{u['name']} -> {u['role']}")
EOF
python ~/pylab/read_json.py
```

`json.loads` parses a string; `json.load` reads from a file object. f-strings provide readable formatting.

---

## Step 6 — Call a public API with requests

```bash
cat > ~/pylab/api_call.py <<'EOF'
"""Call a public test API and print selected fields."""
import json
import requests


def fetch(url: str) -> dict:
    resp = requests.get(url, timeout=10)
    resp.raise_for_status()
    return resp.json()


if __name__ == "__main__":
    payload = fetch("https://httpbin.org/get")
    print("URL :", payload["url"])
    print("UA  :", payload["headers"].get("User-Agent"))
    print(json.dumps(payload, indent=2)[:200], "...")
EOF
python ~/pylab/api_call.py
```

`raise_for_status` turns 4xx/5xx responses into exceptions so failures are loud, not silent.

---

## Step 7 — Use a package module and apply PEP 8

```bash
cat > ~/pylab/helpers/__init__.py <<'EOF'
EOF
mkdir -p ~/pylab/helpers
cat > ~/pylab/helpers/__init__.py <<'EOF'
"""Tiny helper package."""
EOF
cat > ~/pylab/helpers/fs.py <<'EOF'
from pathlib import Path


def count_files(root: str) -> int:
    return sum(1 for _ in Path(root).rglob("*") if _.is_file())
EOF

cat > ~/pylab/use_pkg.py <<'EOF'
from helpers.fs import count_files

print("files in /etc:", count_files("/etc"))
EOF

cd ~/pylab && python use_pkg.py
black ~/pylab
flake8 ~/pylab --max-line-length=100
```

`black` rewrites code to a canonical style; `flake8` reports remaining issues. Together they enforce PEP 8 without arguments in code review.

---

## Step 8 — Cleanup

```bash
deactivate || true
rm -rf ~/lab-venv ~/pylab
apt remove -y --purge python3-venv python3-pip
apt autoremove -y
```

This removes the venv, the scripts, and the OS packages installed for the lab.

---

## What you learned
- How to create and use Python virtual environments with `venv` and `pip`.
- The core built-in types: `bool`, `int`, `float`, `str`, `list`, `dict`.
- How to walk a filesystem with `os.walk`, parse JSON, and call a REST API with `requests`.
- How to organise code into a package and apply PEP 8 with `black` and `flake8`.
- Why type hints help long-term maintenance even though Python is dynamically typed.

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- Python — https://docs.python.org/3/
- Requests — https://requests.readthedocs.io/
- Black — https://black.readthedocs.io/
- Flake8 — https://flake8.pycqa.org/
- httpbin — https://httpbin.org/
