# Portable Ansible bundle

This repo is designed to be copied to **any Linux machine** and run from its folder (no `/etc/ansible` layout required).

## Quick start (run on the Linux control node)

```bash
sudo dnf install -y ansible || sudo apt install -y ansible
git clone <this-repo> ansible-bundle
cd ansible-bundle
```

Edit inventory:

- `inventory/hosts`

## Install push-server (standalone mode)

```bash
ansible-playbook -i inventory/hosts push-server.yml
```

By default `push-server.yml` runs with `push_server_configure_web: false` (no nginx/apache/Bitrix integration).

## Install transformer (standalone mode)

```bash
ansible-playbook -i inventory/hosts transformer.yml
```

By default `transformer.yml` runs with `transformer_standalone: true` (packages/services only).

## Notes

- If you **do** have a Bitrix cluster and want integration steps (delegate to `cluster_web_server`, update pool files, etc.),
  run playbooks with `-e push_server_configure_web=true` / `-e transformer_standalone=false` and provide the required
  variables (for example `cluster_web_server`, `web_site_name`, `web_site_dir`, ...).

