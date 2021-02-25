# ns8-scratchpad

NethServer 8 experiments using containers on Fedora 33

## Initialize the control plane

1. Create a [GitHub PAT](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token)
   for the **ghcr.io** registry (private `repo` read-only access should be enough) then run the following command, specifying
   your GitHub user name and providing the generated PAT as password:

       # podman login --auth-file /usr/local/etc/registry.json

2. Execute as root:

       # curl https://raw.githubusercontent.com/DavidePrincipi/ns8-scratchpad/main/control-plane/init.sh | bash init.sh

## Control plane components

The control plane runs the following components:

1. Redis instance running as rootless container of the `cplane` user, TCP port 6379 - host network.

2. `node-agent.service` Systemd unit, running as root. The events are defined in `/usr/local/share/agent/node-events` and `/var/lib/agent/node-events` (for local Sysadmin overrides).

Further components will be added in the future (e.g. API Server, VPN, ...).

Once the control plane has been initialized run this Redis command (replace `fc1` with the output of `hostname -s`) 
to initialize the control plane of the Traefik module instance:

    HSET traefik0/module.env LE_EMAIL davide.principi@nethesis.it EVENTS_IMAGE ghcr.io/nethserver/cplane-traefik:latest
    PUBLISH fc1:module.init traefik0

Access to redis with:

    $ podman run -it --network host --rm redis redis-cli

As alternative

    # dnf install nc
    # nc 127.0.0.1 6379 <<EOF
    HSET traefik0/module.env LE_EMAIL davide.principi@nethesis.it EVENTS_IMAGE ghcr.io/nethserver/cplane-traefik:latest
    PUBLISH fc1:module.init traefik0
    EOF



## Data plane components

- The control plane instantiates a set of *modules* (e.g. Webtop, Nextcloud, OpenLDAP, Traefik...) which constitute
  the data plane.

- An exclusive unix user account is created for each module instance (e.g. `webtop0`, `nextcloud0`, `openldap0`...).

- The unix user account has session lingering enabled: 
  it automatically starts a persistent Systemd user manager.
  The Systemd user process runs the `module-agent.service` unit.

- Each module provides a bunch of event handlers. An event is handled by one or more *action* scripts, stored under `$HOME/.config/module-events`. The local Sysadmin can extend and/or override them by putting their action scripts under `$HOME/module-events`.

- The `module-agent.service` Systemd unit executes the event handlers. The agent daemon runs in the Python virtual
  environment installed in `/usr/local/share/agent/`: action scripts inherit the same environment. Additional binaries
  can be installed under `/usr/local/share/agent/bin/`.

- Each module has granted full read-only access to the Redis database.

- Each module has a public/private key pair to encrypt passwords and other secrets in the Redis database.

### Traefik module

After initializing the control plane, the following Redis command starts the Traefik module instance `traefik0`:

    PUBLISH traefik0:module.init traefik0

The first PUBLISH pulls the module event definitions. The second PUBLISH pulls the service image and runs it.

To inspect and modify the module start a SSH session. SSH is preferred to `su - traefik0` because the latter
does not properly initialize the Systemd session environment.

    # ssh traefik0@localhost
    # systemctl --user status

To uninstall the `traefik0` module run

    # loginctl disable-linger traefik0
    # userdel -r traefik0
