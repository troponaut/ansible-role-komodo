# Ansible Role for Komodo

This role is designed for managing systemd deployments of the komodo periphery agent
trying to minimize the permissions available to the service by creating a service user
and running the systemd service as that user. The user will only have access to:

* It's configuration files
* The periphery agent binary
* It's ssl certificates for establishing a TLS connection with Komodo Core
* It's repo and stacks directories, located in the komodo users home directory

In this way, it should have no more access to the host system than it would running
in a docker container. But since it is running directly on the host filesystem, it should eliminate
the numerous edge cases which appear when running it as a docker container.

## Features

1. **Install** the Komodo Periphery agent, creating and sandboxing the `komodo` user.
2. **Update** the Komodo Periphery agent by specifying a new version.
3. **Uninstall** the Komodo Periphery agent, optionally removing the `komodo` user and home directories.

## Role Variables

Below are some key variables; see [`defaults/main.yml`](./defaults/main.yml) for more details:

- **`komodo_action`**  
  - Controls which operation is performed:
    - `"install"`: Installs the agent fresh
    - `"update"`: Updates the agent to a new version
    - `"uninstall"`: Removes the agent (with optional user removal)
  - *No default*: If unset, the role won’t perform these tasks.

- **`komodo_version`**  
  - The version of Komodo Periphery to install or update.
  - Example: `v1.16.12`

- **`komodo_delete_user`**  
  - If `true` during `uninstall`, removes the `komodo_user` entirely.
  - Default: `false`
  
- **`passkey`**  
  - This must match the passkey set for your Komodo Core install
  - I recommend encrypting with vault (i.e. `ansible-vault encrypt_string 'supersecretpasskey' --name 'passkey'`)

- **`komodo_allowed_ips`**  
  - You should set this to the host IP of Komodo Core, `127.0.0.1` For example, create a host file with the komodo_allowed_ips set there

The remaining variables in [`defaults/main.yml`](./defaults/main.yml) are set to sensible values, but you should
review them and set according to your own needs

You will also need to change the variable for `passkey` to 

## Installation / Setup

1. `ansible-galaxy role install bpbradley.komodo`
2. Create an `inventory/hosts.yml` file which specifies your komodo hosts and indicates the allowed_ips if desired
    ```yaml
    komodo:
        hosts:
            komodo_core_host:
                ansible_host: 192.168.10.20
                komodo_allowed_ips:
                    - "127.0.0.1"
            komodo_periphery_agent1:
                ansible_host: 192.168.10.20
                komodo_allowed_ips:
                    - "192.168.10.20"
    ```
3. **Optional** but recommended. Set an encrypted passkey using `ansible-vault` which matches the passkey set in Komodo Core.

    ```sh
    ansible-vault encrypt_string 'supersecretpasskey' --name 'passkey'
    ```
    You will get an output like this, which we will use later. 

    ```
    passkey: !vault |
            $ANSIBLE_VAULT;1.1;AES256
            65353234373130353539663661376563613539303866643963363830376661316638333139343366
            3563656637303235373336336131346338336634653232300a313736396336316330666237653237
            64613231323433373637313462633863613732653136366462313134393938623136326633346166
            3834333462333162310a313037306336613061313733363862633437376133316234326431633131
            35386565333538623231643433396334323132616438353839663534373030393266
    ```

    Note that you will need to now input the password you entered every time you run this role,
    or you can create a password file for automation.

    ```sh
    echo "your password" > .vault_pass
    chmod 600 .vault_pass
    ```

    Now you can call your playbook with `--vault-password-file .vault_pass`

4. Create a playbook which selects the role. You can create multiple playbooks for install/uninstall/update, or just one
playbook and control behavior with variables. Here is an example of doing it with just one playbook.

    `playbooks/komodo.yml`

    ```yaml
    ---
    - name: Manage Komodo Service
    hosts: komodo
    roles:
        - role: bpbradley.komodo
        komodo_action: "install"
        komodo_version: "v1.16.12"
        passkey: !vault |
            $ANSIBLE_VAULT;1.1;AES256
            65353234373130353539663661376563613539303866643963363830376661316638333139343366
            3563656637303235373336336131346338336634653232300a313736396336316330666237653237
            64613231323433373637313462633863613732653136366462313134393938623136326633346166
            3834333462333162310a313037306336613061313733363862633437376133316234326431633131
            35386565333538623231643433396334323132616438353839663534373030393266
    ```

    Note that you can just enter a cleartext passkey here instead if not using the vault. This playbook will
    default to an install of v1.16.12, but this can be overridden.

5. Run the playbook

    Install using default values

    ```sh
    ansible-playbook -i inventory/komodo.yaml playbooks/komodo.yml \
    --vault-password-file .vault_pass
    ```

    Install an older version instead

    ```sh
    ansible-playbook -i inventory/komodo.yaml playbooks/komodo.yml \
    -e "komodo_version=v1.16.11" \
    -e "komodo_bin=periphery" \
    --vault-password-file .vault_pass
    ```

    Update to the newer version

    ```sh
    ansible-playbook -i inventory/komodo.yaml playbooks/komodo.yml \
    -e "komodo_action=update" \
    -e "komodo_version=v1.16.12" \
    --vault-password-file .vault_pass

    ```

    Uninstall the periphery agent and all installed files, and delete the user.

    ```sh
    ansible-playbook -i inventory/komodo.yaml playbooks/komodo.yml \
    -e "komodo_action=uninstall" \
    -e "komodo_delete_user=true" \
    --vault-password-file .vault_pass
    ```
