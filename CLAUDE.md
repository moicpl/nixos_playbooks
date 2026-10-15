# nixos_playbooks

Entry point for lab1's NixOS machines and the apps in their containers. Use the
`ansible-expert` skill for all Ansible work here and in the two collections.

## Workspace layout

The workspace root is `/root` (also the container work dir). The NixOS work
spans four repos, plus the app state store:

- `nixos_playbooks/` (this repo): thin playbooks only, each importing a
  collection playbook (`ansible.builtin.import_playbook: moicpl.nixos.arr_nix`).
  No logic, no roles of its own. Run playbooks from inside this directory so its
  `ansible.cfg` is picked up. `ansible.cfg` is a local, gitignored copy of
  `ansible_example.cfg`, created by `workspace_setup.yml` when missing.
  - `bootstrap_nix.yml`, `home_management.yml`, `arr_nix.yml` → `moicpl.nixos`
  - `config/<app>.yml` (`frigate`, `sonarr`, `prowlarr`), run as
    `ansible-playbook config/<app>.yml -e app_action=<action>` → `moicpl.app_config`
  - `ssh_keypair.yml` (the SSH key in Bitwarden; a copy of lab1playbooks' one,
    which the legacy playbooks still use) and `appstate_key.yml` (the app
    state store's age key) are local tools, not collection content.
- `collection-nixos/` (https://github.com/moicpl/collection-nixos),
  collection `moicpl.nixos`: installing NixOS machines, their base system and
  their containers. Playbooks `bootstrap_nix`, `home_management`, `arr_nix`
  (`playbooks/`, with the portal templates in `playbooks/templates/`), role
  `moicpl.nixos.nixos`.
- `collection-app-config/` (https://github.com/moicpl/collection-app-config),
  collection `moicpl.app_config`: what runs inside the containers (logins,
  settings, cameras, series). Playbooks `frigate`, `sonarr`, `prowlarr`, each
  with its role `moicpl.app_config.<app>`, with settings from
  `inventory/group_vars/<group>/<app>.yml`. Depends on `moicpl.nixos` (login
  through `connect.yml` of its `nixos` role). Home Assistant and UniFi have
  none: managed by hand.
- `lab1inventory/` (https://github.com/moicpl/lab1inventory), linked as
  `nixos_playbooks/inventory`. Kept outside the collections. Paths in
  playbooks and roles don't depend on the playbook's folder: the inventory is
  `ansible_inventory_sources | first`, never `playbook_dir` (which is inside a
  collection clone).
- `lab1appstate/` (https://github.com/moicpl/lab1appstate), linked as
  `nixos_playbooks/appstate`: the app state store, `<host>/<app>/` per app.
  `appstate_key.yml` creates its age key once (private key `appstate_age_key`
  in Bitwarden, public key and `appstate_dir` in
  `inventory/group_vars/all/appstate.yml`) and its `.sops.yaml`.
- `bitwarden/` (https://github.com/moicpl/bitwarden): the standalone
  `bitwarden` role, shared with the legacy `lab1playbooks`. Not a collection,
  so the collections' playbooks call it by its short name and find it in this
  repo's `roles_path` (`roles/bitwarden`).

`workspace_setup.yml` sets all this up and works from any directory: it asks
for the vault password (`.ansible_vault_pswd`), creates `ansible.cfg`, installs
the galaxy collections of `collections/requirements.yml` into
`~/.ansible/collections`, clones its git collections into the workspace root
and links them as `collections/ansible_collections/moicpl/<name>` (first in
`collections_path`, so the clones are what runs), clones and links the roles of
`roles/requirements.yml`, the inventory and the app state store, and installs
`sops` and `age` into `~/.local/bin`. Edit and commit collection and role code
in their clones, not here. To add a collection or role, add it to its
requirements file and rerun the playbook.

`lab1playbooks/` holds the legacy, non-NixOS playbooks (podman, proxmox,
AWX...). It no longer has anything NixOS.

## Rules

Durable project decisions. Follow them in this repo and both collections, and
point out existing code that breaks them.

1. **Bitwarden Secrets Manager is the only secret store, used only through the
   `bitwarden` role.** It reads and writes secrets by name
   (`bitwarden_action: get|put`, `bitwarden_secret_name`,
   `bitwarden_secret_content`), so no other playbook or role calls the
   `bitwarden.secrets` lookup or the SDK. No plaintext secrets, and no new Ansible
   Vault secrets. The one exception is `bw_lab1_master_token` itself (the
   machine-account access token), see rule 3.
2. **Every secret a playbook needs is read in `pre_tasks`.** Reading them up front
   makes the play fail before it changes anything, does each lookup once instead
   of on every use, and keeps secrets out of roles.
   - Include the `bitwarden` role with `bitwarden_action: get` once per secret
     (a loop is fine), with `tags: always` and `apply: tags: always` so `--tags`
     runs still get them (`tags` alone only tags the include, not the role's tasks). The
     values land in `bitwarden_secrets.<name>` for every host in the play.
   - Other roles never talk to Bitwarden. They take secrets as plain input
     variables and mark tasks that use them with `no_log: true`.

   ```yaml
   pre_tasks:
     - name: <playbook.yml> | Get the secrets from Bitwarden
       ansible.builtin.include_role:
         name: bitwarden
         apply:
           tags: always
       vars:
         bitwarden_action: get
         bitwarden_secret_name: "{{ item }}"
       loop:
         - cloudflare_api_token
       tags: always
   ```
3. **The Bitwarden machine-account token lives in the code, encrypted with
   Ansible Vault.** It is the one secret that can't come from Bitwarden, so it is
   committed as the `bw_lab1_master_token` variable in
   `lab1inventory/group_vars/all/bitwarden.yml`, encrypted with
   `ansible-vault encrypt_string --name bw_lab1_master_token`. It is never
   committed in plaintext and never replaced by an environment variable or a file
   outside the repos. The vault password is never committed. It lives only in
   `nixos_playbooks/.ansible_vault_pswd` (mode 0600, gitignored), which
   `workspace_setup.yml` creates by asking for it.
4. **Every task name starts with its file name.** In every playbook and role
   task file, name each task `<file name>.yml | <Action> <description>`,
   so the run output shows which file each task comes from.

   ```yaml
   # roles/nginx/tasks/install.yml
   - name: install.yml | Install nginx package
   ```
5. **`inventory/group_vars/all/bitwarden.yml` holds only Bitwarden auth and
   connection settings:** the token, `bw_api_url`, `bw_identity_url`,
   `bw_organization_id` and `bw_project_id`, defined nowhere else. Secret IDs are
   not Bitwarden settings: each one is `<secret name>_bws_id` in the vars file of
   its own topic (e.g. the SSH key IDs in `ssh.yml`, next to the key settings).
   The `bitwarden` role records the ID of each secret it creates in the file
   given as `bitwarden_secret_id_file`. This keeps one topic per file.
6. **Installed NixOS machines accept SSH keys only, and root works only on the
   console.** `PermitRootLogin no` and no password or keyboard-interactive login,
   so a password can't open a remote session. Each machine gets its own random
   root password, stored in Bitwarden as `<host>_root_password` (dots as `_`)
   before the machine uses it. The live installer's root password
   (`nixos_bootstrap_root_password`) is typed at the `bootstrap_nix.yml` prompt,
   used only for that install and never stored.
7. **`nixos*` names are only the placeholder of the NixOS live installer** (USB
   stick, hostname `nixos`), never a machine's name. `bootstrap_nix.yml` always
   asks for the real hostname and adds it to `[nixos]`. When the target answers
   as `nixos*`, the root login with the password entered works and the erase
   switch (`nixos_install_wipe`) is on, it erases the disk, whatever the machine
   holds.
8. **NixOS machines are set up in layers, by separate playbooks.**
   `bootstrap_nix.yml` installs a machine and keeps the base system of
   every machine in `[nixos]`: SSH key login, root password, network (DHCP or a
   fixed address, asked once and kept in its host_vars), base packages. An apps
   playbook (`home_management.yml`, `arr_nix.yml`...) only adds containers,
   their ports and secrets, on its own group (`nixos_home`, `nixos_arr`), a child
   of `[nixos]`; a new machine is moved there by hand. So the apps never touch
   the base, and one playbook never touches another's machines. These two
   layers are `moicpl.nixos`. What runs inside the containers (settings, users,
   cameras) is a third layer, `moicpl.app_config` (rule 9), which never changes
   containers.
9. **Each app's state has its own playbook, `moicpl.app_config.<app>` (run as
   `config/<app>.yml`), with an `app_action` switch:** `configure` sets the app
   to the desired state, `backup` saves the live state in the app state store,
   `default` resets the app to factory state (with only the admin login from
   Bitwarden where the app allows it). The desired state is code from the
   inventory wherever the app accepts it (Frigate's `config.yaml`, the Sonarr
   and Prowlarr APIs); only what can't be code is restored from stored files.
   `default` erases data, so it runs only with `-e app_wipe=true`; whatever
   erases or replaces app data first keeps a snapshot on the host
   (`/var/lib/appstate/<app>/snapshots`).
10. **Stored app state lives only in the app state store (`lab1appstate`), and
    any file that can hold a secret is encrypted there with sops to the store's
    age key.** The private key is `appstate_age_key` in Bitwarden (rule 1), the
    public key `appstate_age_public_key` in the inventory. Plain text only for
    files meant to hold no secrets. Encryption and decryption
    happen in memory on the controller, which never writes a decrypted file.
11. **Logic lives in the collections; this repo only imports their playbooks.**
    Roles and playbooks inside a collection call each other by FQCN
    (`moicpl.nixos.nixos`, `moicpl.app_config.sonarr`); only the standalone
    `bitwarden` role is called by its short name. Nothing in a collection
    writes next to its playbooks (`playbook_dir`): that is inside the clone.
    This keeps the collections installable and runnable from any project.

## Maintaining this file

When the user states a new lasting convention or decision ("always...", "never...",
"from now on..."), add it to **Rules** as a numbered entry with a short reason, or
update the rule it changes. Keep entries short and don't duplicate them. Remove a rule
only when the user retires it. Update **Workspace layout** when the structure changes.
