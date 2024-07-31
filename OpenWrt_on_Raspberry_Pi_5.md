# OpenWrt on Raspberry Pi 5

A portable router to use on the go, based on [OpenWrt](https://openwrt.org) and Raspberry Pi 5.

Network adapters:
- *Dell USB-C to RJ45 DBQBCBC064 Gigabit Ethernet Adapter*
- *Netgear Nighthawk AXE3000 USB 3.0 WiFi Adapter (A8000)*

# Preparations

Change Wireless country code and update EEPROM from inside Raspberry Pi OS (see [this](https://openwrt.org/toh/raspberry_pi_foundation/raspberry_pi)).

Download OpenWrt [FACTORY (SQUASHFS)](https://firmware-selector.openwrt.org/?version=SNAPSHOT&target=bcm27xx%2Fbcm2712&id=rpi-5) snapshot image. Flash the image, insert the microSD card into Raspberry Pi and power on.

Double check the output of `uci` before running `service` or `reboot` commands - no errors should be displayed.

# Initial setup

Change password for `root` after first boot.

```sh
=== WARNING! =====================================
There is no root password defined on this device!
Use the "passwd" command to set up a new password
in order to prevent unauthorized SSH logins.
--------------------------------------------------
root@OpenWrt:/# passwd
Changing password for root
New password:
Retype password:
passwd: password for root changed by root
```

## Configure system

```sh
uci set system.@system[0].hostname='rocinante.lan'
uci set system.@system[0].description='When in Rome'
uci set system.@system[0].zonename='Europe/Bucharest'
uci set system.@system[0].timezone='EET-2EEST,M3.5.0/3,M10.5.0/4'

uci delete system.ntp.server

uci add_list system.ntp.server='0.pool.ntp.org'
uci add_list system.ntp.server='1.pool.ntp.org'
uci add_list system.ntp.server='2.pool.ntp.org'
uci add_list system.ntp.server='3.pool.ntp.org'

uci commit system

service system restart
```

## Configure network <a id="lan"/>

Create a profile containing router IP and bitmask:

```sh
mkdir /etc/profile.d
cat << "EOF" > /etc/profile.d/$(uname -n).sh
alias ss=netstat

export router_lan_ip=172.24.42.65
export router_lan_bitmask=27
export router_dhcp_min=67
export router_dhcp_max=93
eval $(resize) # RS232
EOF

source /etc/profile.d/$(uname -n).sh
```

Then configure LAN and WAN:

```sh
uci delete network.lan.netmask

uci set network.lan.ipaddr="${router_lan_ip}/${router_lan_bitmask}"
uci set network.lan.force_link=1

uci set network.wan=interface
uci set network.wan.proto='dhcp'
uci set network.wan.peerdns='0'
uci set network.wan.dns='1.1.1.3 1.0.0.3'

uci commit network
```

Update DHCP range:

```sh
uci set dhcp.lan.start=${router_dhcp_min}
uci set dhcp.lan.limit=${router_dhcp_max}

uci commit dhcp

service network restart
```

## Configure wireless (STA) <a id="sta0"/>

```sh
uci set wireless.radio0.channel='auto'
uci set wireless.radio0.disabled='0'
uci set wireless.radio0.country='RO'

uci rename wireless.default_radio0='sta0'

uci set wireless.sta0.mode='sta'
uci set wireless.sta0.network='wan'
uci set wireless.sta0.ssid='Tycho Station'
uci set wireless.sta0.encryption='psk2'
uci set wireless.sta0.key='"[...], so long as you are on the right side."'

uci commit wireless

wifi
```

Both `phy0-sta0` and `br-lan` interfaces should now have IPs assigned from an existing Wi-Fi router (the one broadcasting the `ssid` configured above) and from the `lan` [configuration](#lan), respectively.

Test internet connection on the router:

```sh
opkg update
```

## Install and configure the web UI

```sh
opkg update
opkg install luci-ssl-nginx
```

### Change the default certificate

Existing private and public key files:

```sh
/etc/nginx/conf.d/_lan.key
/etc/nginx/conf.d/_lan.crt
```

### Reconfigure NGINX

```sh
uci set nginx._lan.uci_manage_ssl='no' # do not allow LuCI to manage certificates
uci set nginx._lan.server_name=$(uname -n)

uci -q delete nginx._lan.listen
uci -q delete nginx._redirect2ssl.listen

uci add_list nginx._lan.listen="${router_lan_ip}:443 ssl"
uci add_list nginx._redirect2ssl.listen="${router_lan_ip}:80"

uci commit nginx

service nginx restart
```

Check configured IP and ports:

```sh
root@rocinante:/# ss -tln | grep -E ':80|:443'
tcp        0      0 172.24.42.65:443          0.0.0.0:*               LISTEN
tcp        0      0 172.24.42.65:80           0.0.0.0:*               LISTEN
```

### Reconfigure SSH

```sh
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NEE5AAAAIMd7CmaALG3anehAM8" > /etc/dropbear/authorized_keys

uci set dropbear.@dropbear[0].Interface='lan'
uci set dropbear.@dropbear[0].PasswordAuth='off'
uci set dropbear.@dropbear[0].RootPasswordAuth='off'

uci commit dropbear

service dropbear restart
```

Check configured IP and port:

```sh
root@rocinante:/# ss -tln | grep :22
tcp        0      0 172.24.42.65:22        0.0.0.0:*               LISTEN
```

# Router configuration

The router will be configured with the internal Wi-Fi in mode `sta` - the device providing the internet -, and an external Wi-Fi adapter connected via USB in mode `ap` - the device external clients will connect to. It will use the built-in Raspberry Pi ethernet adapter (`eth0`) as a WAN connection and delegate its LAN membership to the USB ethernet adapter.

The `sta` device configuration will change whenever location changes, the `ap` device configuration will remain the same.

## Base packages

```sh
opkg update
opkg install kmod-usb-core kmod-usb3
```

## Optional packages

```sh
opkg update
opkg install usbutils pciutils ip-full grep vim-runtime vim-full git-http jq
```

## The Ethernet Adapter

### Install kernel drivers

```sh
opkg update
opkg install kmod-usb-net-rtl8152
```

### Configure adapter

Insert the ethernet adapter in one of the two USB 3.0 ports of the Raspberry Pi.

Change the LAN bridge `br-lan` to include `eth1` instead of `eth0`:

```sh
uci set network.@device[0].ports='eth1'

uci commit network

service network restart
```

Connect an ethernet cable between a computer and Raspberry Pi via the USB adapter. The router should now be ready for connections over SSH and HTTPS, using its configured IP address (the one specified in the `router_lan_ip` environment variable).

## The Wi-Fi Adapter (AP)

This Wi-Fi adapter does not support VAPs (multiple SSIDs).

### Install kernel drivers

```sh
opkg update
opkg install kmod-mt7921u
```

### Configure adapter

Connect the adapter; its details should automatically be added in `/etc/config/wireless`.

Configure AP details:

```sh
uci rename wireless.radio2='radio1'
uci rename wireless.default_radio2='ap0'

uci set wireless.radio1.band='5g'
uci set wireless.radio1.channel='auto'
uci set wireless.radio1.htmode='HE80'
uci set wireless.radio1.disabled='0'

uci set wireless.ap0.device='radio1'

uci set wireless.ap0.ssid='Rocinante'
uci set wireless.ap0.encryption='sae'
uci set wireless.ap0.key='Do 3 what 3 Romans 0 Do 1'

uci commit wireless

wifi
```

Connect a client device to the `ssid` specified above, you should be able to reach the internet.

## Using the ethernet port

Connect the router to Internet using the ethernet port [instead of its Wi-Fi radio](#sta0):

```sh
uci set network.wan.device='eth0'

uci commit network

service network restart
```

Revert to Wi-Fi:

```sh
uci delete network.wan.device

uci commit network

service network restart
```

# Miscellaneous

## Firewall

```sh
uci set firewall.@defaults[0].drop_invalid='1'

uci commit firewall

service firewall restart
```

## Packet steering

```sh
uci set network.globals.packet_steering='2'
uci set network.globals.steering_flows='128'

uci commit network

service network restart
```

### Autostart AP<sup>*</sup>

```sh
cat << "EOF" > /usr/libexec/autostart-radio1
#!/bin/sh

IFS='-' read -r _ radio <<_EOF
$(basename "$0")
_EOF

status=$(wifi status | jq ".${radio}.up")
if [ "${status}" = 'false' ]; then
  wifi
fi

exit 0
EOF
chmod 755 /usr/libexec/autostart-radio1
echo "*/1 * * * * /usr/libexec/autostart-radio1" > /etc/crontabs/root
```

<sup>*)</sup> Ugly, but I don't know how to activate the AP after a reboot.

### Expand filesystem<sup>**</sup>

Install required packages:

```sh
opkg update
opkg install cfdisk squashfs-tools-unsquashfs losetup resize2fs 
```

Check `DEVICE` and `PARTITION` are correct.

```sh
DEVICE="/dev/mmcblk0"   # SD card
cfdisk "$DEVICE"        # resize, then write
PARTITION="${DEVICE}p2" # should be the Linux (83) partition
  
FS_SIZE="$(unsquashfs -s "$PARTITION" | grep -o 'Filesystem size [0-9]* bytes' | grep -o '[0-9][0-9]*')"
FS_OFFSET="$(expr '(' "$FS_SIZE" + 65535 ')' / 65536 '*' 65536)" 
LOOP_DEVICE="$(losetup -f --show -o "$FS_OFFSET" "$PARTITION")"

# fsck, resize, fsck
e2fsck -f $LOOP_DEVICE
# e2fsck 1.47.0 (5-Feb-2023)
# rootfs_data: recovering journal
# rootfs_data primary superblock features different from backup, check forced.
# Pass 1: Checking inodes, blocks, and sizes
# Pass 2: Checking directory structure
# Pass 3: Checking directory connectivity
# /lost+found not found.  Create<y>? yes
# Pass 4: Checking reference counts
# Pass 5: Checking group summary information
# Feature orphan_present is set but orphan file is clean.
# Clear<y>? yes

# rootfs_data: ***** FILE SYSTEM WAS MODIFIED *****
# rootfs_data: 880/25376 files (0.5% non-contiguous), 31938/101696 blocks

resize2fs $LOOP_DEVICE
# resize2fs 1.47.0 (5-Feb-2023)
# Resizing the filesystem on /dev/loop1 to 31088448 (1k) blocks.
# The filesystem on /dev/loop1 is now 31088448 (1k) blocks long.

e2fsck -f $LOOP_DEVICE
# e2fsck 1.47.0 (5-Feb-2023)
# rootfs_data: clean, 880/7407840 files, 1888214/31088448 blocks

reboot
```

<sup>**)</sup> When using [SquashFS](https://en.wikipedia.org/wiki/SquashFS) only.

## Simple Ad blocker


```sh
cat << "EOF" > /usr/libexec/get-block-list
#!/bin/sh

l1="https://v.firebog.net/hosts/AdguardDNS.txt"
l2="https://raw.githubusercontent.com/DandelionSprout/adfilt/master/Alternate%20versions%20Anti-Malware%20List/AntiMalwareHosts.txt"

hosts_file="/etc/adblock.hosts"

interval=$(( 24 * 60 * 60 ))

update='yes'
if [ -f "${hosts_file}" ]; then
  file_size=$(du -k ${hosts_file} | awk '{ print $1 }' )
  if [ ${file_size} -gt 0 ]; then
    last_modified_seconds=$(date -r ${hosts_file} +%s)
    seconds_lapsed=$(( $(date +%s) - ${last_modified_seconds} ))

    if [ ${seconds_lapsed} -lt ${interval} ]; then
      update='no'
    fi
  fi
fi

if [ "${update}" = 'yes' ]; then
  wget -q -O - "${l1}" "${l2}" 2> /dev/null | grep -E '^[^#|^!]' | awk -F' ' '!a[$NF]++ {gsub(/^/,"0.0.0.0 ",$NF) ; print $NF ; gsub(/^0\.0\.0\.0/,"::1",$NF) ; print $NF}' > ${hosts_file} && service dnsmasq restart > /dev/null
fi

exit 0
EOF
chmod 755 /usr/libexec/get-block-list
echo "*/10 * * * * /usr/libexec/get-block-list" >> /etc/crontabs/root

/usr/libexec/get-block-list
```

Configure `dnsmasq`:

```sh
uci set dhcp.@dnsmasq[0].interface='lan'
uci set dhcp.@dnsmasq[0].addnhosts='/etc/adblock.hosts'

uci commit dhcp

service dnsmasq restart

```

## Kernel drivers for other adapters

### Wi-Pi 11n 150 Mbps Wi-Fi USB

VAP support: yes.

```sh
kmod-rt2800-lib kmod-rt2800-usb kmod-rt2x00-lib kmod-rt2x00-usb
```

### TP Link TL-WN821N 11n 300 Mbps (V1)

VAP support: yes.

```sh
kmod-ath9k-htc
```

### D-Link DUB 1312 USB 3.0 Gigabit Ethernet Adapter

```sh
kmod-usb-net-asix-ax88179
```

# References

- [OpenWrt Table of Hardware](https://openwrt.org/toh/start)
- [OpenWrt Raspberry Pi](https://openwrt.org/toh/raspberry_pi_foundation/raspberry_pi)
- [OpenWrt Dropbear configuration](https://openwrt.org/docs/guide-user/base-system/dropbear)
- [OpenWrt Network Configuration](https://openwrt.org/docs/guide-user/network/network_configuration)
- [OpenWrt The UCI system](https://openwrt.org/docs/guide-user/base-system/uci)
- [Linux in-kernel USB WiFi Adapters](https://github.com/morrownr/USB-WiFi/blob/main/home/USB_WiFi_Adapters_that_are_supported_with_Linux_in-kernel_drivers.md)
- [OpenWrt Firmware Selector](https://firmware-selector.openwrt.org/?version=SNAPSHOT&target=bcm27xx%2Fbcm2712&id=rpi-5)
- [OpenWrt SD card](https://openwrt.org/docs/guide-user/installation/installation_methods/sd_card)
- [Dnsmasq as a Network-Wide Adblocker](https://jears.at/jblog/dnsmasq_adblock.jblog)

