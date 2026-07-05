# Day 72 -- Ansible Project: Automate Docker and Nginx Deployment

## Task
Combine everything learned across five days of Ansible -- inventory, playbooks, modules, handlers, variables, facts, conditionals, loops, roles, templates, Galaxy, and Vault -- into one complete project. Automate a full deployment: install Docker, pull and run a containerized app, set up Nginx as a reverse proxy in front of it, and manage everything through Ansible roles. One command takes a fresh server to a fully running, production-style setup.

## Task 1: Project Structure
Built the complete layout with `ansible-galaxy init` for each role:
```
ansible-docker-project/
  ansible.cfg
  inventory.ini
  site.yml
  .vault_pass          # gitignored
  group_vars/
    all.yml
    web/
      vault.yml         # encrypted Docker Hub credentials
  roles/
    common/
      tasks/main.yml
    docker/
      tasks/main.yml
      handlers/main.yml
      defaults/main.yml
    nginx/
      tasks/main.yml
      templates/
        app-proxy.conf.j2
      handlers/main.yml
      defaults/main.yml
```

## Task 2: Common Role
Runs on every host -- updates package cache, installs baseline packages (vim, curl, wget, git, htop, tree, jq, unzip), sets timezone, and creates the deploy user.

## Task 3: Docker Role
Installs Docker CE and its dependencies, starts and enables the service, adds the deploy user to the docker group, logs into Docker Hub using Vault-encrypted credentials, pulls the app image, and runs the container with `restart_policy: always`, mapping port 8080 on the host to port 80 in the container. A health check with the `uri` module confirms the container responds before moving on.

## Task 4: Nginx Role
Installs Nginx, removes the default site config, deploys the reverse proxy config from a Jinja2 template pointing at `127.0.0.1:8080`, runs `nginx -t` to validate config before reloading, and starts/enables the service. A handler reloads Nginx only when the config changes.

## Task 5: Vault
Docker Hub username and password stored encrypted in `group_vars/web/vault.yml` via `ansible-vault create`. A `.vault_pass` file lets `ansible-playbook` run without prompting, and it's excluded from Git through `.gitignore`. Secrets never appear in plain text anywhere in the project.

## Task 6: Master Playbook and Deploy
`site.yml` runs three tagged plays -- `common` on all hosts, `docker` and `nginx` on the `web` group.

```bash
ansible-playbook site.yml -i inventory.ini
```

Result: `ok=18, changed=2, failed=0`

Verified with:
```bash
docker ps
curl localhost:8080
curl localhost:80
```

Container confirmed running:
```
CONTAINER ID   IMAGE          PORTS                  NAMES
8f8dd2dd70c9   nginx:latest   0.0.0.0:8080->80/tcp   myapp
```
`curl localhost:8080` hit the container directly, `curl localhost:80` came back through Nginx's reverse proxy -- confirming the full chain works.

Selective runs used tags:
```bash
ansible-playbook site.yml -i inventory.ini --tags docker
ansible-playbook site.yml -i inventory.ini --tags nginx
ansible-playbook site.yml -i inventory.ini --skip-tags common
```

## Task 7: Idempotency Check
Re-ran the full playbook a second time -- came back with no unnecessary changes, proving the entire stack is idempotent from a fresh run.

## Architecture
```
Client → Nginx:80 → Docker Container (nginx:latest):8080
```

## Key Takeaway
Everything from Day 68 through Day 72 came together here: inventory and SSH setup, playbooks and handlers, variables and conditionals, roles and Vault -- all combined into one command that takes a server from zero to a running, secured, production-style deployment.

`#90DaysOfDevOps` `#DevOpsKaJosh` `#TrainWithShubham`
