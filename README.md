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
  - For example:

    ```yaml
    passkey: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          65353234373130353539663661376563613539303866643963363830376661316638333139343366
          3563656637303235373336336131346338336634653232300a313736396336316330666237653237
          64613231323433373637313462633863613732653136366462313134393938623136326633346166
          3834333462333162310a313037306336613061313733363862633437376133316234326431633131
          35386565333538623231643433396334323132616438353839663534373030393266
    ```

- **`komodo_allowed_ips`**  
  - You should set this to the host IP of Komodo Core, `127.0.0.1` For example, create a host file with the allowed_ips set there
  - For example, an `inventory/komodo.yml` might look like:
    
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

The remaining variables in [`defaults/main.yml`](./defaults/main.yml) are set to sensible values, but you should
review them and set according to your own needs

You will also need to change the variable for `passkey` to 
