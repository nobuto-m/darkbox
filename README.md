## Install

Install Ubuntu Server with the latest LTS (focal, 20.04 as of writing).
https://releases.ubuntu.com/focal/

With:
- The default storage layout, "Use an entire disk" and "Set up this disk as an LVM group".
- Install OpenSSH server and import a key.

## Packages

Install the following packages:
```
anacron
avahi-daemon
etckeeper
libvirt-daemon-system
uvtool-libvirt
prometheus-node-exporter
wireguard-tools
```

### Third-party

https://tailscale.com/download/linux/ubuntu-2004


## Configuration

### LVM

Extend the root LV to use the entire drive, LP: #1785321 and LP: #1893276.

```bash
$ sudo lvresize /dev/ubuntu-vg/ubuntu-lv -l +100%FREE --resizefs -v
```

### sudoers

Set up `NOPASSWD` since it's a testbed.

```bash
$ cat <<EOF | sudo tee /etc/sudoers.d/99-local
$USER ALL=(ALL) NOPASSWD:ALL
EOF
```

### Wake on LAN

Enable Wake on LAN explicitly since `WakeOnLan=` in systemd.link is off
by default.

```diff
diff --git a/netplan/00-installer-config.yaml b/netplan/00-installer-config.yaml
index dea791e..68f62ea 100644
--- a/netplan/00-installer-config.yaml
+++ b/netplan/00-installer-config.yaml
@@ -2,5 +2,8 @@
 network:
   ethernets:
     enp31s0:
+      match:
+        macaddress: 70:85:c2:ae:bc:08
       dhcp4: true
+      wakeonlan: true
   version: 2
```

### LXD

Use the latest stable one instead of pre-installed one.

```bash
$ sudo snap refresh --channel latest/stable lxd
```


```
cat <<EOF | sudo lxd init --preseed
cluster: null
networks:
- config:
    ipv4.address: 10.0.9.1/24
    ipv4.nat: "true"
    ipv4.dhcp.ranges: 10.0.9.51-10.0.9.200
    ipv6.address: none
  description: ""
  managed: true
  name: lxdbr0
  type: ""
  project: default
storage_pools:
- config: {}
  description: ""
  name: default
  driver: dir
profiles:
- config: {}
  description: ""
  devices:
    eth0:
      name: eth0
      network: lxdbr0
      type: nic
    root:
      path: /
      pool: default
      type: disk
  name: default
EOF
```

squid-deb-proxy container

```
cat <<EOF | lxc init ubuntu:focal --config=user.user-data="$(cat /dev/stdin)" squid-deb-proxy
#cloud-config

packages:
  - anacron
  - squid-deb-proxy

write_files:
  - content: |
      ppa.launchpad.net

      ## apt-add-repository
      launchpad.net
      keyserver.ubuntu.com

      images.maas.io
      artifacts.elastic.co

    owner: root:root
    path: /etc/squid-deb-proxy/mirror-dstdomain.acl.d/50-thirdparty
    permissions: '0644'

runcmd:
  - systemctl mask --now squid
  - printf '\nshutdown_lifetime 3 seconds\n' >> /etc/squid-deb-proxy/squid-deb-proxy.conf
EOF
```

static ip for squid-deb-proxy and launch

```
lxc network attach lxdbr0 squid-deb-proxy eth0 eth0
lxc config device set squid-deb-proxy eth0 ipv4.address 10.0.9.2

lxc start squid-deb-proxy
```

apply squid-deb-proxy globally

```
cat <<EOF | lxc profile set default user.vendor-data "$(cat /dev/stdin)"
#cloud-config
apt_proxy: http://squid-deb-proxy.lxd:8000/
EOF
```
