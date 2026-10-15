# nixos_playbooks

Runs lab1's NixOS machines and the apps in their containers. The playbooks here
only import those of two collections:

- [moicpl.nixos](https://github.com/moicpl/collection-nixos): install, base
  system, containers
- [moicpl.app_config](https://github.com/moicpl/collection-app-config): the
  apps' state inside the containers

The inventory is [lab1inventory](https://github.com/moicpl/lab1inventory), the
app state store [lab1appstate](https://github.com/moicpl/lab1appstate).

## Setup

```sh
git clone https://github.com/moicpl/nixos_playbooks.git
ansible-playbook nixos_playbooks/workspace_setup.yml   # interactive: asks for the vault password
cd nixos_playbooks
```

It clones the collections, the `bitwarden` role, the inventory and the app
state store next to this repo and links them here.

## Playbooks

| Command | What it does |
| --- | --- |
| `ansible-playbook bootstrap_nix.yml [-e nixos_install_wipe=true]` | Install a machine from the live installer; base system of all `[nixos]` machines |
| `ansible-playbook home_management.yml` | Containers of `[nixos_home]` (UniFi, Home Assistant, Frigate) |
| `ansible-playbook arr_nix.yml` | Containers of `[nixos_arr]` (Sonarr, Radarr, Lidarr, Prowlarr, Transmission, Jellyfin, Immich) |
| `ansible-playbook config/<app>.yml -e app_action=configure\|backup\|default` | State of Frigate, Sonarr or Prowlarr |
| `ansible-playbook ssh_keypair.yml` | Create the SSH key pair in Bitwarden (once) |
| `ansible-playbook appstate_key.yml` | Create the app state store's age key (once) |
