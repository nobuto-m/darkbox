
add universe back, and set security.ubuntu.com, LP: #1783129

    sudo sed -i -e 's/ main$/\0 universe/' /etc/apt/sources.list
    sudo sed -i -e 's|//.*\(/ubuntu .*-security \)|//security.ubuntu.com\1|' /etc/apt/sources.list


ddclient

    sudo apt install ddclient

```
NOIP_USERNAME=
NOIP_PASSWORD=
NOIP_HOSTNAME=

cat <<EOF | sudo tee /etc/ddclient.conf
use=web
ssl=yes

protocol=noip
login=$NOIP_USERNAME
password=$NOIP_PASSWORD
$NOIP_HOSTNAME
EOF
```

resize /

    sudo lvresize /dev/ubuntu-vg/ubuntu-lv -L 80G --resizefs -v


LXD setup

    sudo apt install thin-provisioning-tools

    sudo lvcreate -L 300G --thin --discard nopassdown ubuntu-vg -n LXDThinPool

```
cat <<EOF | sudo lxd init --preseed
config:
  images.auto_update_interval: "0"
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
storage_pools:
- config:
    source: ubuntu-vg
    lvm.thinpool_name: LXDThinPool
    volume.block.mount_options: noatime,nobarrier,data=writeback
  description: ""
  name: default
  driver: lvm
profiles:
- config: {}
  description: ""
  devices:
    eth0:
      name: eth0
      nictype: bridged
      parent: lxdbr0
      type: nic
    root:
      path: /
      pool: default
      type: disk
  name: default
EOF
```

cat <<EOF | lxc init ubuntu:bionic --config=user.user-data="$(cat /dev/stdin)" squid-deb-proxy
#cloud-config

packages:
  - anacron
  - squid-deb-proxy

write_files:
  - content: |
      ppa.launchpad.net
      images.maas.io
      artifacts.elastic.co
    owner: root:root
    path: /etc/squid-deb-proxy/mirror-dstdomain.acl.d/50-thirdparty
    permissions: '0644'

run_cmd:
  - service squid-deb-proxy restart
EOF



lxc network attach lxdbr0 squid-deb-proxy eth0 eth0
lxc config device set squid-deb-proxy eth0 ipv4.address 10.0.9.2

lxc start squid-deb-proxy

cat <<EOF | lxc profile set default user.user-data "$(cat /dev/stdin)"
#cloud-config
apt_proxy: http://squid-deb-proxy.lxd:8000/
EOF
