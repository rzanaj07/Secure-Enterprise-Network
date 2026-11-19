# 5. Wazuh SIEM on Ubuntu Server

> **Status:** ⏳ Planned. Screenshots will be added when this phase is complete.

Wazuh is an open-source SIEM and XDR platform. The all-in-one deployment runs the **manager** (analysis engine), the **indexer** (OpenSearch-based storage), and the **dashboard** (web UI) on one Ubuntu Server VM in the Security segment (`ether4-LAN Security`, 192.168.30.0/24).

## 5.1 Static IP (netplan)

`/etc/netplan/50-cloud-init.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [192.168.30.10/24]
      routes:
        - to: default
          via: 192.168.30.1
      nameservers:
        addresses: [192.168.20.10, 8.8.8.8]
```

```bash
sudo netplan apply
```

## 5.2 All-in-one install

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The installer prints the dashboard `admin` password at the end. Save it and **don't** commit it.

Open the dashboard at `https://192.168.30.10`.

## 5.3 Receive MikroTik syslog

Add this to `/var/ossec/etc/ossec.conf` on the manager:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>192.168.30.1</allowed-ips>
</remote>
```

```bash
sudo systemctl restart wazuh-manager
```
