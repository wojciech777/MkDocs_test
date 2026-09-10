# Install on a host

Orion Platform 4.2.1 is available as `.deb` and `.rpm` packages for deployments without containers, on physical or virtual machines. The packages install the component binaries, the systemd units and a configuration skeleton in `/etc/orion`. The method requires the Business or Enterprise license variant.

## Adding the repository and installing

=== "Ubuntu 22.04 / 24.04 (.deb)"

    ```bash
    curl -fsSL https://charts.nimbus.example.com/apt/nimbus.gpg \
      | sudo tee /usr/share/keyrings/nimbus.gpg > /dev/null

    echo "deb [signed-by=/usr/share/keyrings/nimbus.gpg] \
    https://charts.nimbus.example.com/apt orion-4.2 main" \
      | sudo tee /etc/apt/sources.list.d/nimbus-orion.list

    sudo apt-get update
    sudo apt-get install -y \
      orion-gateway=4.2.1 \
      orion-core=4.2.1 \
      orion-ledger=4.2.1 \
      orion-worker=4.2.1 \
      orion-scheduler=4.2.1 \
      orion-console=4.2.1
    ```

    Pinning the version against an accidental upgrade:

    ```bash
    sudo apt-mark hold orion-gateway orion-core orion-ledger \
      orion-worker orion-scheduler orion-console
    ```

=== "RHEL 9 (.rpm)"

    ```bash
    sudo tee /etc/yum.repos.d/nimbus-orion.repo > /dev/null <<'EOF'
    [nimbus-orion]
    name=Nimbus Orion 4.2
    baseurl=https://charts.nimbus.example.com/rpm/el9/orion-4.2
    enabled=1
    gpgcheck=1
    gpgkey=https://charts.nimbus.example.com/rpm/nimbus.asc
    EOF

    sudo dnf install -y \
      orion-gateway-4.2.1 \
      orion-core-4.2.1 \
      orion-ledger-4.2.1 \
      orion-worker-4.2.1 \
      orion-scheduler-4.2.1 \
      orion-console-4.2.1
    ```

    Excluding the packages from automatic updates:

    ```bash
    sudo dnf versionlock add 'orion-*'
    ```

!!! note "Components can be split across machines"
    The packages are independent. A typical production split is two machines with `orion-gateway` and `orion-console`, two with `orion-core` and `orion-ledger`, and two with `orion-worker` and `orion-scheduler`. All of them must point at the same `orion` database and the same Kafka brokers.

## Directory layout

```text
/etc/orion/
├── orion.yaml                 main configuration
├── orion.dev.yaml             dev profile overrides (optional)
├── conf.d/
│   ├── 10-database.yaml
│   ├── 20-kafka.yaml
│   └── 30-observability.yaml
└── secrets/
    ├── db-password
    ├── webhook-signing-secret
    └── encryption-key

/var/lib/orion/
├── cache/                     local component cache
├── spool/                     buffer for outgoing events
└── tmp/                       temporary files of ledger exports

/var/log/orion/
├── gateway.log
├── core.log
├── ledger.log
├── worker.log
└── scheduler.log

/usr/lib/systemd/system/
├── orion-gateway.service
├── orion-core.service
├── orion-ledger.service
├── orion-worker.service
├── orion-scheduler.service
└── orion-console.service

/usr/bin/orion                 command line tool
```

The packages create an `orion` system user without a login shell. The `/etc/orion/secrets` directory has mode `0750` and owner `root:orion`.

## The systemd unit

```ini title="/usr/lib/systemd/system/orion-core.service" hl_lines="10 11 21"
[Unit]
Description=Orion Platform - orders engine (orion-core)
Documentation=https://docs.orion.example.com
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=orion
Group=orion
EnvironmentFile=-/etc/orion/orion-core.env
Environment=ORION_CONFIG=/etc/orion/orion.yaml
Environment=ORION_PROFILE=prod
ExecStartPre=/usr/bin/orion config validate --config /etc/orion/orion.yaml
ExecStart=/usr/bin/orion-core serve --listen 0.0.0.0:8081
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s
TimeoutStopSec=30s
WorkingDirectory=/var/lib/orion
StateDirectory=orion
LogsDirectory=orion
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/orion /var/log/orion
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

!!! tip "Do not edit files under /usr/lib/systemd/system"
    A package upgrade overwrites the units. Put your own changes in the override directory:

    ```bash
    sudo systemctl edit orion-core
    ```

## Starting the services

```bash
sudo orion db migrate --config /etc/orion/orion.yaml

sudo systemctl enable --now \
  orion-core orion-ledger orion-worker orion-scheduler orion-gateway orion-console

systemctl status orion-core --no-pager
```

Expected output:

```text
● orion-core.service - Orion Platform - orders engine (orion-core)
     Loaded: loaded (/usr/lib/systemd/system/orion-core.service; enabled)
     Active: active (running) since Tue 2025-11-04 09:41:12 CET; 3min 8s ago
       Docs: https://docs.orion.example.com
   Main PID: 24817 (orion-core)
      Tasks: 38 (limit: 18841)
     Memory: 612.4M (peak: 688.1M)
        CPU: 9.412s
     CGroup: /system.slice/orion-core.service
             └─24817 /usr/bin/orion-core serve --listen 0.0.0.0:8081

Nov 04 09:41:12 orion-app-01 orion-core[24817]: schema core=4.2.1 ledger=4.2.1 audit=4.2.1
Nov 04 09:41:13 orion-core[24817]: kafka producer ready topic=orion.events.v1
Nov 04 09:41:13 orion-app-01 systemd[1]: Started orion-core.service.
```

Verifying the API:

```bash
curl -s http://localhost:8080/healthz
```

Logs from the systemd journal:

```bash
journalctl -u orion-core -f --since "10 min ago"
```

## Upgrading the version

!!! danger "Take a backup before upgrading"
    Schema migrations are irreversible. Before every upgrade, take a full dump of the `orion` database and a copy of the `/etc/orion` directory following the procedure in [Backups](../../operations/backups.md). Without a backup there is no way back to the previous version.

1. Review the list of changes for the target release in the [Changelog](../../changelog.md).
2. Take a database dump and archive the configuration:
   ```bash
   sudo -u postgres pg_dump -Fc orion > /var/backups/orion-$(date +%F).dump
   sudo tar czf /var/backups/orion-etc-$(date +%F).tgz /etc/orion
   ```
3. Turn off inbound traffic on port 8080 at the load balancer or firewall.
4. Stop the components in reverse dependency order:
   ```bash
   sudo systemctl stop orion-console orion-gateway orion-scheduler orion-worker orion-ledger orion-core
   ```
5. Remove the version pin and install the new packages:

    === "Ubuntu 22.04 / 24.04 (.deb)"

        ```bash
        sudo apt-mark unhold 'orion-*'
        sudo apt-get update && sudo apt-get install -y 'orion-*=4.2.1'
        sudo apt-mark hold 'orion-*'
        ```

    === "RHEL 9 (.rpm)"

        ```bash
        sudo dnf versionlock delete 'orion-*'
        sudo dnf upgrade -y 'orion-*'
        sudo dnf versionlock add 'orion-*'
        ```

6. Compare your configuration with the new template, which the package writes as `/etc/orion/orion.yaml.dpkg-dist` or `.rpmnew`:
   ```bash
   sudo orion config validate --config /etc/orion/orion.yaml
   ```
7. Run the schema migrations:
   ```bash
   sudo orion db migrate --config /etc/orion/orion.yaml
   ```
8. Start the components in dependency order:
   ```bash
   sudo systemctl start orion-core orion-ledger orion-worker orion-scheduler orion-gateway orion-console
   ```
9. Confirm `"version": "4.2.1"` in the `GET /healthz` response and restore inbound traffic.
10. Watch `orion_queue_lag` and `orion_worker_retries_total` for the first 30 minutes after the upgrade.

??? info "Uninstalling"
    Removing the packages does not remove the data or the configuration:

    ```bash
    sudo systemctl disable --now 'orion-*'
    sudo apt-get purge 'orion-*'      # or: sudo dnf remove 'orion-*'
    ```

    The `/var/lib/orion` and `/etc/orion/secrets` directories stay on disk. Delete them manually once you confirm they are no longer needed.

## See also

- [Installation](index.md)
- [Configuration](../configuration/index.md)
- [Backups](../../operations/backups.md)
- [Logs](../../operations/monitoring/logs.md)
