# WiFi connectivity

### Requirements

Just in case you don't have these commands, you should install them beforehand:

    # pacman -S wpa_supplicant dhcpcd

I am also asumming there is no trail of `NetworkManager` and `dhclient` packages. 

### WPA Supplicant

The package `wpa_supplicant` can configure network interfaces and connect to wireless networks. It is intended to run as a daemon and let other commands to connect it. 

1. Find the name of your wireless network interface, it usually starts with `wl`. For this tutorial we will use `wlp1s0`

       # ip l
    
2. Generate the file `/etc/wpa_supplicant/wpa_supplicant-wlp1s0.conf` by using the utility `wpa_passphrase`, which is also part of the `wpa_supplicant` package

       # wpa_passphrase your-SSID your-PASS | sudo tee /etc/wpa_supplicant/wpa_supplicant-wlp1s0.conf
       
Open the ouput file and be sure it looks similar to this one (add anything that's missing)

       ctrl_interface=/run/wpa_supplicant
       update_config=1
       
       network={
           ssid="your-SSID"
           psk=somegeneratedpsk
       }
       
3. At this point we could connect to the wireless network with:

       # wpa_supplicant -B -c /etc/wpa_supplicant-wlp1s0.conf -i wlp1s0

4. The idea is to configure `wpa_supplicant` to allow WiFi connections at system startup, in order to do so:

       # systemctl enable wpa_supplicant@wlp1s0.service

Now we are connected to the wireless network but we still need an IP to be assigned via DHCP.

### DHCP Client Daemon

1. In order to get an IP automatically after connecting to the WiFi we need enable the respective service:

       # systemctl enable dhcpcd@wlp1s0.service
       
If everything went ok, we should be able to connect automatically to the specified WiFi after the next reboot.

### Based on

* [Arch Linux dhcpcd](https://wiki.archlinux.org/index.php/dhcpcd#Installation)
* [Arch Linux wpa_supplicant](https://wiki.archlinux.org/index.php/Wpa_supplicant#Connecting_with_wpa_passphrase)
* [LinuxBabe](https://www.linuxbabe.com/command-line/ubuntu-server-16-04-wifi-wpa-supplicant)
