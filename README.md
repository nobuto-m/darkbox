
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
