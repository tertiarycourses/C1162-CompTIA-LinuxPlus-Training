# Lab 20 — Infrastructure as Code with Ansible

This lab introduces Infrastructure as Code (IaC) using Ansible, the most common agentless configuration management tool covered by CompTIA Linux+ XK0-006 Objective 4.1 (Given a scenario, implement and use common infrastructure as code, automation, and orchestration tools). You will also see brief examples of Puppet, OpenTofu (Terraform fork), cloud-init, and Kubernetes ConfigMaps so you can recognise each ecosystem on the exam.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Install Ansible and prepare a working directory

```bash
apt update
apt install -y ansible
mkdir -p /root/lab && cd /root/lab
ansible --version
```

Ansible is shipped in the Ubuntu universe repository. The working directory `/root/lab` will hold the inventory, playbook, and templates.

---

## Step 2 — Create an inventory pointing at localhost

```bash
cat > /root/lab/inventory.ini <<'EOF'
[web]
localhost ansible_connection=local
EOF

ansible -i /root/lab/inventory.ini all -m ping
```

The `ansible_connection=local` setting tells Ansible to run modules directly on this machine without SSH. The `ping` module confirms connectivity.

---

## Step 3 — Run ad-hoc commands

```bash
cd /root/lab
ansible -i inventory.ini all -m apt -a "name=tree state=present" -b
ansible -i inventory.ini all -m command -a "tree -L 1 /etc"
```

The `-b` flag is "become" (sudo). Ad-hoc commands are great for one-off tasks; for repeatable work, write a playbook.

---

## Step 4 — Write a playbook with a variable, template, and handler

```bash
cat > /root/lab/site.yml <<'EOF'
- name: Deploy a simple nginx site
  hosts: web
  become: true
  vars:
    site_title: "Linux+ XK0-006"
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: true

    - name: Deploy index page from template
      template:
        src: index.html.j2
        dest: /var/www/html/index.html
      notify: Reload nginx

  handlers:
    - name: Reload nginx
      service:
        name: nginx
        state: reloaded
EOF

cat > /root/lab/index.html.j2 <<'EOF'
<h1>{{ site_title }}</h1>
<p>Managed by Ansible on {{ ansible_hostname }}.</p>
EOF

ansible-playbook -i inventory.ini site.yml --check
ansible-playbook -i inventory.ini site.yml
curl -s localhost | head
```

`--check` is a dry run. Handlers run only when notified by a changed task, which is how Ansible avoids unnecessary service restarts.

---

## Step 5 — Install a Galaxy collection and peek at other IaC tools

```bash
ansible-galaxy collection install community.general
ansible-galaxy collection list | head

# Puppet (declarative, agent-based alternative)
apt install -y puppet
puppet apply -e 'notice("hello from puppet")'

# OpenTofu (open-source Terraform fork)
mkdir -p /root/lab/tofu && cd /root/lab/tofu
cat > main.tf <<'EOF'
resource "null_resource" "hello" {
  provisioner "local-exec" {
    command = "echo hello-from-opentofu"
  }
}
EOF
apt install -y unzip wget
wget -q https://github.com/opentofu/opentofu/releases/download/v1.8.5/tofu_1.8.5_linux_amd64.zip
unzip -o tofu_1.8.5_linux_amd64.zip tofu -d /usr/local/bin/ && chmod +x /usr/local/bin/tofu
tofu init && tofu apply -auto-approve
cd /root/lab
```

Ansible Galaxy hosts community collections (extra modules). Puppet uses a declarative DSL; OpenTofu provisions cloud resources through providers.

---

## Step 6 — GitOps, cloud-init, and Kubernetes ConfigMap samples

```bash
cat > /root/lab/cloud-init.yaml <<'EOF'
#cloud-config
package_update: true
packages:
  - nginx
write_files:
  - path: /var/www/html/index.html
    content: "Provisioned by cloud-init"
runcmd:
  - systemctl enable --now nginx
EOF

cat > /root/lab/configmap.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  SITE_TITLE: "Linux+ XK0-006"
  LOG_LEVEL: "info"
EOF

ls /root/lab
```

GitOps means storing these declarative files in Git so a controller (Argo CD, Flux) reconciles the cluster to match. `cloud-init` runs on first boot of cloud VMs; ConfigMaps decouple config from container images in Kubernetes.

---

## Step 7 — Cleanup

```bash
ansible -i /root/lab/inventory.ini all -m apt -a "name=nginx state=absent purge=yes" -b
apt remove -y --purge ansible puppet tree
rm -rf /root/lab /usr/local/bin/tofu
apt autoremove -y
```

This removes the packages and working files so the playground is fresh for the next exercise.

---

## What you learned
- How to write an Ansible inventory, ad-hoc command, and idempotent playbook with variables, templates, and handlers.
- How to install community collections via `ansible-galaxy`.
- How Puppet, OpenTofu, cloud-init, and Kubernetes ConfigMaps fit into the wider IaC landscape.
- The GitOps pattern of storing declarative config in Git as the source of truth.

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- Ansible — https://docs.ansible.com/
- Puppet — https://www.puppet.com/docs/puppet/
- OpenTofu — https://opentofu.org/
- cloud-init — https://cloudinit.readthedocs.io/
- Kubernetes docs — https://kubernetes.io/docs/concepts/configuration/configmap/
