# Lab 24 — Responsible AI Use for Linux Administrators

This lab is the hands-on companion to CompTIA Linux+ XK0-006 Objective 4.5 (Given a scenario, summarise the use of AI in a Linux environment). The focus is the verify-before-paste workflow: how to prompt an LLM for a Linux command, review the answer, lint and test it locally, redact secrets before sending data to a hosted model, and choose between local and hosted models based on policy. You will also draft a short corporate AI usage policy.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Set up a verification workspace

```bash
apt update && apt install -y shellcheck jq curl
mkdir -p /root/ai-lab && cd /root/ai-lab
shellcheck --version | head -1
```

Every AI-generated shell snippet you accept should pass `shellcheck` first. `jq` and `curl` let you talk to JSON APIs.

---

## Step 2 — Try a local LLM (optional, skip if it fails)

```bash
# Optional: install Ollama and a small CPU-friendly model.
# Killercoda has no GPU and limited RAM, so this may be slow or fail.
curl -fsSL https://ollama.com/install.sh | sh || echo "Ollama install skipped"
(ollama serve >/tmp/ollama.log 2>&1 &) ; sleep 2
ollama pull tinyllama 2>/dev/null && \
  ollama run tinyllama "Write a one-line bash command to list the 5 largest files in /var/log" \
  || echo "Local model unavailable on this playground; continue with the placeholder API workflow."
```

Local models keep prompts on your machine, which matters when prompts contain sensitive data. If this step fails on the playground, that is fine — continue with the next step.

---

## Step 3 — Hosted-API placeholder workflow (no real key needed)

```bash
# We simulate a hosted LLM by hitting a free echo service.
# In real life, replace the URL with your provider's endpoint and
# pass the API key via an env var, never hard-coded in the script.
export LLM_API_KEY="REDACTED-EXAMPLE"

cat > /root/ai-lab/ask.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
prompt="${1:-Write a bash one-liner to show top 5 largest files under /var/log}"
# Redact obvious secrets before sending.
safe_prompt=$(printf '%s' "$prompt" \
  | sed -E 's/(AKIA[0-9A-Z]{16})/REDACTED-AWS-KEY/g' \
  | sed -E 's/(ghp_[A-Za-z0-9]{20,})/REDACTED-GH-TOKEN/g' \
  | sed -E 's/(password=)[^ ]+/\1REDACTED/gi')

curl -s -X POST https://httpbin.org/post \
  -H "Authorization: Bearer ${LLM_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg p "$safe_prompt" '{prompt:$p}')" \
  | jq '.json.prompt'
EOF
chmod +x /root/ai-lab/ask.sh
/root/ai-lab/ask.sh "list AKIAABCDEFGHIJKLMNOP and password=hunter2"
```

Notice the script masks AWS access keys, GitHub tokens, and `password=` strings before the prompt leaves the machine. The API key is read from an env var, never written in the source.

---

## Step 4 — Verify an AI-suggested command

```bash
# Pretend the LLM returned this snippet. Save and inspect it BEFORE running.
cat > /root/ai-lab/suggested.sh <<'EOF'
#!/usr/bin/env bash
# AI-suggested: top 5 largest files under /var/log
find /var/log -type f -printf '%s %p\n' 2>/dev/null \
  | sort -nr | head -5 | awk '{printf "%10d  %s\n",$1,$2}'
EOF
chmod +x /root/ai-lab/suggested.sh

shellcheck /root/ai-lab/suggested.sh
cat /root/ai-lab/suggested.sh    # read it line by line
/root/ai-lab/suggested.sh        # only run after review
```

The verification rule is simple: read every line, run `shellcheck`, then execute in a non-production environment first. Never paste an AI command directly into a production root shell.

---

## Step 5 — Build a verification checklist

```bash
cat > /root/ai-lab/checklist.md <<'EOF'
# Verify-before-paste checklist

- [ ] Did I read every line of the suggested command/script?
- [ ] Did I run `shellcheck` (or `pylint`/`mypy` for Python)?
- [ ] Does the command touch /, /etc, /var, /boot, or any production path?
- [ ] Does it use `rm -rf`, `dd`, `mkfs`, `chmod 777`, or pipe to `sh`?
- [ ] Are there hard-coded credentials, tokens, or internal hostnames?
- [ ] Have I tested it in a sandbox (Killercoda, VM, container)?
- [ ] Is the output what I actually expected?
- [ ] Have I attributed AI assistance per company policy?
EOF
cat /root/ai-lab/checklist.md
```

Treat this list as gating criteria. If any box is unchecked, do not run the snippet on a real system.

---

## Step 6 — Draft a corporate AI usage policy

```bash
cat > /root/ai-lab/policy.md <<'EOF'
# AI Usage Policy (template)

## 1. Scope
Applies to all engineers using generative AI (hosted or local) for code,
configuration, scripts, or incident response on company systems.

## 2. Permitted models
- Local models (Ollama, llamafile) on company-owned hardware: allowed for
  prompts containing internal hostnames, log excerpts, or non-secret config.
- Hosted models (OpenAI, Anthropic, etc.) on an approved enterprise tier:
  allowed only after secrets/PII have been redacted.
- Personal/free hosted tiers: NOT allowed for any internal data.

## 3. Data handling
- No customer PII, credentials, API keys, private keys, or proprietary
  source code may be sent to any model without DPO approval.
- Use the redaction script in the team toolkit before any hosted call.
- Treat all model outputs as untrusted input until reviewed.

## 4. Verification
- All AI-generated shell or Python must pass `shellcheck`/`flake8` and be
  reviewed line-by-line before execution outside a sandbox.
- Destructive commands require a peer review.

## 5. Training-data governance
- Do not opt in to provider training on company prompts.
- Prefer providers with documented zero-retention or short-retention modes.

## 6. Attribution
- Note AI assistance in commit messages or PR descriptions when it shaped
  the change materially.

## 7. Incidents
- Suspected secret leakage to a model: revoke the credential immediately
  and notify security within 1 hour.
EOF
wc -l /root/ai-lab/policy.md
```

This is a starting template. Real policies should be reviewed by legal, security, and the data protection officer before adoption.

---

## Step 7 — Compare local vs hosted at a glance

```bash
cat <<'EOF'
Local model (Ollama, llamafile)
  + Prompts and data never leave the host
  + Works offline; no per-token cost
  - Smaller models, lower quality
  - You manage GPU/CPU and updates

Hosted model (OpenAI/Anthropic/etc.)
  + Highest-quality models, fast
  + No infra to maintain
  - Prompts traverse the internet to a third party
  - Cost per token; possible training/retention concerns
EOF
```

The exam expects you to know that local models trade quality for privacy, while hosted models do the opposite, and that corporate policy decides which fits each task.

---

## Step 8 — Cleanup

```bash
unset LLM_API_KEY
rm -rf /root/ai-lab
# Stop and remove Ollama if it was installed
pkill -f "ollama serve" 2>/dev/null || true
rm -rf /usr/local/bin/ollama /usr/share/ollama ~/.ollama 2>/dev/null || true
apt remove -y --purge shellcheck jq
apt autoremove -y
```

This removes the working files, the env var, and any local model artifacts.

---

## What you learned
- A repeatable verify-before-paste workflow: read, `shellcheck`, sandbox, then run.
- How to redact secrets (AWS keys, GitHub tokens, passwords) before sending prompts to a hosted model.
- The trade-offs between local and hosted LLMs for system administration tasks.
- How to structure a corporate AI usage policy covering scope, data handling, verification, training-data governance, attribution, and incident response.

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- ShellCheck — https://www.shellcheck.net/
- jq — https://jqlang.github.io/jq/
- Ollama (optional) — https://ollama.com/
- httpbin (API placeholder) — https://httpbin.org/
