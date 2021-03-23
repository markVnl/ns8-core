# ns8-scratchpad

NethServer 8 experiments

System requirements: Systemd, Podman: both Fedora 33 and Debian 10 were used in the tests.


## Core components

The core purpose is managing the applications, providing the basics for their entire lifecycle (install, upgrade, reconfigure, uninstall...). It runs the following components:

1. Redis instance running as rootfull container, bound to TCP port 6379. The Redis DB
   stores the system and modules configuration and provides a signaling bus based on its PUB/SUB feature.

2. `node-agent.service` Systemd unit, running as root. The events are defined in `/usr/local/share/agent/node-events`
   and `/var/local/node-events` (for local Sysadmin overrides).

3. `module-agent.service` Systemd units, running in each module as non-privileged users. See the "Additional modules" section below for more details.

4. Edge proxy, for TLS termination and centralized certificates management (Traefik)

5. LDAP proxy, listening on 127.0.0.1 port 3890. It helps other modules to connect to the account provider LDAP service,
   providing a fixed address and clear text connection.

6. LDAP local account provider (OpenLDAP, Samba DC)


Further components will be added in the future (e.g. API Server, VPN, ...).

## Applications

- The core instantiates a set of *modules*. Each module instance
  runs an application (e.g. Webtop, Nextcloud) as a set of one or more Podman **rootless containers**. In exceptional
  cases a module can run also rootfull containers; the only known exception by now is Samba DC.

- An exclusive unix user account is created for each module instance (e.g. `webtop0`, `nextcloud0`, `openldap0`...).

- The unix user account has session lingering enabled: it automatically starts a persistent Systemd user manager instance.

- A module is installed by the core `node-agent` service, when it receives a "module.init" event.
  The next sections show some examples of the Redis HSET/PUB commands that signal that event.

- A module provides a bunch of event handlers and Systemd unit definitions.
  An event is handled by one or more *action* scripts, stored under `$HOME/.config/module-events`. 
  The local Sysadmin can extend and/or override them by putting their action scripts under `$HOME/module-events`.
  Module-local systemd units are installed under `$HOME/.config/systemd/user`. System-wide units are installed under
  `/etc/systemd/user`.

- A module must provide a `module.init` event implementation that installs and configures
  the Podman containers. Installed services must be enabled to start on boot.

- The `module-agent.service` Systemd unit executes the event handlers. The agent daemon runs in the Python virtual
  environment installed in `/usr/local/share/agent/`: action scripts inherit the same environment. Additional binaries
  can be installed under `/usr/local/share/agent/bin/`.

- Each module has granted full read-only access to the Redis database.

- Each module has a public/private key pair to encrypt passwords and other secrets in the Redis database.

## Core installation

Execute as root:
```
# curl https://raw.githubusercontent.com/DavidePrincipi/ns8-scratchpad/main/core/install.sh | bash
```

When installing on Debian 10 Buster, first make sure to have the latest running kernel:
```
apt-get update
apt-get upgrade -y
reboot
```

If you're a developer and you need to push images to the registry, you must configure the authentication.
Create a [GitHub PAT](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token)
for the **ghcr.io** registry (for read-only access `read:packages private` scope should be enough) then run the following command, specifying
your GitHub user name and providing the generated PAT as password:
```
# podman login --authfile /usr/local/etc/registry.json ghcr.io
```

The core is composed also by the following components:

- traefik, running with `traefik0` user
- restic rest-server, running with user `restic0`


### Redis

Once the core has been initialized, you can access Redis with one of the following command:

    podman run -i --network host --rm docker.io/redis:6-alpine redis-cli <<EOF
    PING
    EOF

As alternative, use `nc` command:

    # nc 127.0.0.1 6379 <<EOF
    ...
    EOF

Or even shorter in Bash:

    # cat >/dev/tcp/127.0.0.1/6379 <<EOF
    ...
    EOF

### Traefik

To inspect and modify the module start a SSH session. SSH is preferred to `su - traefik0` because the latter
does not properly initialize the Systemd session environment. Check the services are running with:

    # ssh traefik0@localhost
    # systemctl --user status

To uninstall the `traefik0` module run

    # loginctl disable-linger traefik0
    # userdel -r traefik0

#### Default Let's Encrypt certificate

To request a Let's Encrypt certificate for the server FQDN, just execute:
```
N=default HOST=$(hostname -f); podman run -i --network host --rm docker.io/redis:6-alpine redis-cli <<EOF
SET traefik/http/routers/$N-http/service $N
SET traefik/http/routers/$N-http/entrypoints http,https
SET traefik/http/routers/$N-http/rule "Host(\`$HOST\`)"
SET traefik/http/routers/$N-https/entrypoints http,https
SET traefik/http/routers/$N-https/rule "Host(\`$HOST\`)"
SET traefik/http/routers/$N-https/tls true
SET traefik/http/routers/$N-https/service $N
SET traefik/http/routers/$N-https/tls/certresolver letsencrypt
SET traefik/http/routers/$N-https/tls/domains/0/main $HOST
EOF
```

Traefik will generate the certificate without exposing any new service.

### Ldapproxy

The LDAP account provider service can be local or remote, and can require TLS or not. To help modules using the LDAP
service, a LDAP proxy is always available at 127.0.0.1:3890 without TLS.

Rootless containers can establish a connection from their private network to the loopback interface with the
following Podman arguments:

    --network=slirp4netns:allow_host_loopback=true --add-host=accountprovider:10.0.2.2

To create a Ldapproxy instance run the following commands, adjusting the LDAPHOST value if needed.

```
cat >/dev/tcp/127.0.0.1/6379 <<EOF
HSET module/ldapproxy0/module.env EVENTS_IMAGE ghcr.io/nethserver/ldapproxy:latest PROXYPORT 3890 LDAPHOST 127.0.0.1 LDAPPORT 636 LDAPSSL on
PUBLISH $(hostname -s):module.init ldapproxy0
EOF
```

### Nsdc

The Nsdc module runs a singleton and rootfull Samba 4 DC instance.

- *Rootfull* because Samba needs special privileges to store ACLs in the filesystem extended attributes
- *Singleton* because Samba services are bound to a host IP address to serve LAN clients, and 127.0.0.1

Initialize the Redis DB and start the installation with:

```
podman run -ti --network host --rm docker.io/redis:6-alpine redis-cli <<EOF
HSET module/nsdc0/module.env EVENTS_IMAGE ghcr.io/nethserver/nsdc:latest NSDC_IMAGE ghcr.io/nethserver/nsdc:latest IPADDRESS 10.133.0.5 HOSTNAME nsdc1.$(hostname -d) NBDOMAIN AD REALM AD.$(hostname -d | tr a-z A-Z) ADMINPASS Nethesis,1234
PUBLISH $(hostname -s):module-rootfull.init nsdc0
EOF
```

The DC storage is persisted to the following Podman local volumes:

- nsdc0-data
- nsdc0-config

## Uninstall

The `core/uninstall.sh` script attempts to stop and erase core components and
additional modules. Handle it with care because it erases everything under `/home/*`!

    bash uninstall.sh

## Prototype validation

List of things considered almost stable, with or without an existing prototype implementation:

- Systemd & Podman foundations
- Unix users for rootless modules
- Wireguard VPN among nodes
- Account providers:
  - Samba AD and OpenLDAP account providers (both are LDAP)
  - Remote LDAP account provider
  - No Unix accounts for domain users
- Node agent / Module agents
- Events and Actions
- Container Registry as software repository for everything
- Store environment variables for actions and containers in Redis
- Authenticated Redis access for write operations
- Public Redis read only access
- Encrypted secrets in Redis DB
- Edge proxies with TLS termination
- Centralized certificate management
- FHS compliancy 
- ...

Still uncertain, undefined:

- Multi-node join/leave promote/demote procedures
- Account provider local proxy
- Multi-master LDAP replication
- Local sysadmin override for Actions and container images
- Report back events state from Agents to API server
- Port allocation for module instances
- Module instance upgrade/downgrade rollout
- API server
- Use Redis accounts to access API server too
- ...

## VPN

Each node is connected to the master node using WireGuard VPN in a star network topology.
After the installation, the server will be configured as master node.

The VPN uses the fixed private network `10.5.4.2.0/24`. The first node will be the master and has the default IP address set to `10.5.4.1`.
All other worker nodes will have IP address like `10.5.4.2`, `10.5.4.3`, etc.

Retrieve the server public key, it will be used inside the next command:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli HGET node/$(hostname -s)/vpn PUBLIC_KEY
```

To add a new node to the VPN, install everything on a new machine and execute:
```
private_key=$(wg genkey)
public_key=$(echo $private_key | wg pubkey)
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli <<EOF
HSET node/$(hostname -s)/vpn PUBLIC_KEY $public_key 
HSET node/$(hostname -s)/vpn IP_ADDRESS <vpn_client_ip>
HSET node/$(hostname -s)/vpn SERVER_PUBLIC_ADDRESS <server_public_address:port>
HSET node/$(hostname -s)/vpn SERVER_PUBLIC_KEY <server_public_key>
PUBLISH $(hostname -s):vpn-worker.init $private_key
EOF
```

Then, get the new node publick key, execute on the worker:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli HGET node/$(hostname -s)/vpn PUBLIC_KEY
```

Access the server and execute: 
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli <<EOF
HSET node/$(hostname -s)/vpn/client1 PUBLIC_KEY <worker_public_key> IP_ADDRESS <vpn_client_ip>
PUBLISH $(hostname -s):vpn-add-client client
EOF
```

## Applications installation

### Dokuwiki

To start a dokuwiki instance execute:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli <<EOF
HSET module/dokuwiki0/module.env EVENTS_IMAGE ghcr.io/nethserver/dokuwiki:latest
PUBLISH $(hostname -s):module.init dokuwiki0
EOF
```

Setup traefik routes:
```
N=dokuwiki HOST=dokuwiki.$(hostname -f); podman run -i --network host --rm docker.io/redis:6-alpine redis-cli <<EOF
SET traefik/http/services/$N/loadbalancer/servers/0/url http://127.0.0.1:8080
SET traefik/http/routers/$N-http/service $N
SET traefik/http/routers/$N-http/entrypoints http,https
SET traefik/http/routers/$N-http/rule "Host(\`$HOST\`)"
SET traefik/http/routers/$N-https/entrypoints http,https
SET traefik/http/routers/$N-https/rule "Host(\`$HOST\`)"
SET traefik/http/routers/$N-https/tls true
SET traefik/http/routers/$N-https/service $N
SET traefik/http/routers/$N-https/tls/certresolver letsencrypt
SET traefik/http/routers/$N-https/tls/domains/0/main $HOST
EOF
```

### Nextcloud

To start a nextcloud instance execute:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli <<EOF
HSET module/nextcloud0/module.env EVENTS_IMAGE ghcr.io/nethserver/nextcloud:latest
PUBLISH $(hostname -s):module.init nextcloud0
EOF
```

Setup traefik:
```
N=nextcloud HOST=nextcloud.$(hostname -f); podman run -i --network host --rm docker.io/redis:6-alpine redis-cli <<EOF
SET traefik/http/services/$N/loadbalancer/servers/0/url http://127.0.0.1:8181
SET traefik/http/routers/$N-http/service $N
SET traefik/http/routers/$N-http/entrypoints http,https
SET traefik/http/routers/$N-http/rule "Host(\`$HOST\`)"
SET traefik/http/routers/$N-https/entrypoints http,https
SET traefik/http/routers/$N-https/rule "Host(\`$HOST\`)"
SET traefik/http/routers/$N-https/tls true
SET traefik/http/routers/$N-https/service $N
SET traefik/http/routers/$N-https/tls/certresolver letsencrypt
SET traefik/http/routers/$N-https/tls/domains/0/main $HOST  
EOF
```

Execute `occ` command:
```
ssh nextcloud0@localhost
podman exec -ti --user www-data nextcloud-app php occ
```

Setup nsdc account provider:
```
ssh nextcloud0@localhost
./scripts/setup_ad.sh
```

Note: the nsdc must have a user named `ldapservice` with password `Nethesis,1234`

### Mail

Installation

```
cat >/dev/tcp/127.0.0.1/6379 <<EOF
HSET module/mail0/module.env HOSTNAME mail.example.com BINDDN %n@AD.DP.NETHSERVER.NET EVENTS_IMAGE ghcr.io/nethserver/mail
PUBLISH $(hostname -s):module.init mail0
EOF
```

TODO

- No TLS config
- No Rspamd, Unbound and Redis, no ClamAV and Olefy
- No OpenDKIM: can we use Rspamd instead?
- The Dovecot static userdb lookups are always successful: attempts to deliver mail to unexisting accounts are always allowed! In
  ns7 the "passwd" userdb (sssd) was implemented to find existing accounts. Here we have to lookup users in the LDAP DB.


## Backup & restore

### First configuration

The core and each instance can implement the `backup` event. The event takes the name of the backup as first argument.

First, generate a password for the new backup and set restic repository base URL:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli SETNX backup/<backup_name>/password <password>
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli SET backup/<backup_name>/base_repository rest:http://127.0.0.1:8383
```

Example:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli SETNX backup/backup1/password $(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1)
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli SET backup/backup1/base_repository rest:http://127.0.0.1:8383
```

### Backup and restore the core

To backup the core execute:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli PUBLISH <hostname>:backup <backup_name>
```

Example:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli PUBLISH $(hostname -s):backup backup1
```

### Restore the core

To restore the core execute:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli PUBLISH $(hostname -s):restore backup1
```

Example:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli PUBLISH $(hostname -s):restore backup1
```

### Backup an instance

To execute the backup of an instance execute:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli PUBLISH <user_instance>:backup <backup_name>
```

Example:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli PUBLISH nextcloud0:backup backup1
```

### Restore an instance

Each instance implementing the `backup` event, must implement also the `restore` event. The event takes the name of the backup as first argument.

To execute the restore of an instance execute:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli PUBLISH <user_instance>:restore <backup_name>
```

Example:
```
podman run -i --network host --rm docker.io/redis:6-alpine redis-cli PUBLISH nextcloud0:restore backup1
```


