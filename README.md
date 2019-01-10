sudoers with nopasswd since it's just a testbed

cat <<EOF | sudo tee /etc/sudoers.d/99-local
$USER ALL=(ALL) NOPASSWD:ALL
EOF


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

    sudo lvresize /dev/ubuntu-vg/ubuntu-lv -L 8G --resizefs -v


LXD setup

```
sudo apt install thin-provisioning-tools
sudo lvcreate -L 300G --thin ubuntu-vg -n LXDThinPool

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

squid-deb-proxy container

```
cat <<EOF | lxc init ubuntu:bionic --config=user.user-data="$(cat /dev/stdin)" squid-deb-proxy
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

run_cmd:
  - service squid-deb-proxy restart
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
cat <<EOF | lxc profile set default user.user-data "$(cat /dev/stdin)"
#cloud-config
apt_proxy: http://squid-deb-proxy.lxd:8000/
EOF
```

Juju
```
sudo snap install juju --classic
sudo snap install juju-wait --classic
sudo snap install juju-crashdump --classic
sudo snap install openstackclients --classic
juju bootstrap --model-default apt-http-proxy="http://squid-deb-proxy.lxd:8000/" localhost
```

juju-openstack model profile

```
lxc profile create juju-openstack

cat <<EOF | lxc profile edit juju-openstack
name: juju-default
config:
  boot.autostart: "true"
  security.nesting: "true"
  security.privileged: "true"
  linux.kernel_modules: openvswitch,nbd,ip_tables,ip6_tables
devices:
  root:
    path: /
    pool: default
    type: disk
  eth0:
    name: eth0
    nictype: bridged
    parent: lxdbr0
    type: nic
  eth1:
    name: eth1
    nictype: bridged
    parent: lxdbr0
    type: nic
  kvm:
    path: /dev/kvm
    type: unix-char
  mem:
    path: /dev/mem
    type: unix-char
  tun:
    path: /dev/net/tun
    type: unix-char
EOF
```


mirror

sudo apt install apt-mirror

cat <<EOF | sudo tee /etc/apt/mirror.list
set nthreads 5
set mirror_path /srv/mirror

deb http://archive.ubuntu.com/ubuntu bionic main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu bionic-security main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu bionic-updates main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu bionic-proposed main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu bionic-backports main restricted universe multiverse

clean http://archive.ubuntu.com/ubuntu
EOF
