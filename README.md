#WIP
# Raspberry-pi-VPN
How to set up your Raspberry Pi zero to act like a portable VPN!
## AP Configuration
1. Install `hostpad` and `dnsmasq`:
```
sudo apt-get update
sudo apt-get install hostapd dnsmasq
```
2. Edit configuration of the `hostpd` located at `/etc/hostapd/hostapd.conf`
```
interface=wlan0
driver=nl80211
ssid=RaspberryPiVPN
channel=2
wpa=2
wpa_passphrase=MyPassPhrase
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP
rsn_pairwise=CCMP
```
2. Edit configuration of the `dnsmasq` located at `/etc/dnsmasq.conf`
```
interface=wlan0
dhcp-range=10.10.0.2,10.10.0.100,12h
server=10.10.0.1
```
3. Set a static IP for the interface:
  * Using `dhcpd` (This did not work in my case..) by editing confic file in `/etc/dhcpcd.conf`:
  ```
  interface=wlan0
  static ip_address=10.10.0.1/24
  nohook wpa_supplicant
  ```
  * Using `systemd` to create a service that runs after startup:
    * Create a bash script under `/root/.local/bin/assign_ip.sh`
       ```
        #!/bin/bash
        ifconfig wlan0 10.10.0.1
        ```
    * Make it executable:
      ```
        chmod u+x /root/.local/bin/assign_ip.sh
        ```
    * Create the service file `/lib/systemd/system/startup.service`:
      ```
        [Unit]
        Description=Startup Script
        
        [Service]
        ExecStart=/root/.local/bin/assign_ip.sh
        
        [Install]
        WantedBy=multi-user.target
        ```
    * Enable the service:
      ```
        systemctl enable startup.service --now
        ```
  ```
  interface=wlan0
  static ip_address=10.10.0.1/24
  nohook wpa_supplicant
  ```


5. Enable and start the services:
```
sudo systemctl enable hostapd
sudo systemctl enable dnsmasq
sudo systemctl start hostapd
sudo systemctl start dnsmasq
```
Now the Raspberry Pi should create an access point with the name that you chose and you can connect to it! But nothing much happening since it's not linked to any internet connection.
Let's change that.
## Internet providing interface Configuration 
In this section, we will set up a secondary wireless card as our way to the internet. Connecting the network card, the Raspberry Pi gives it `wlan1` name

1. Connect the interface to your local wifi (your home network)
2. Enable IP forwarding by uncommenting the line below in the file `/etc/sysctl.conf`:
```
net.ipv4.ip_forward=1
```
3. Apply changes:
```
sudo sysctl -p
```
4. Set up NAT:
```
sudo iptables -t nat -A POSTROUTING -o wlan1 -j MASQUERADE
sudo iptables -A FORWARD -i wlan1 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o wlan1 -j ACCEPT
```
5. Save `iptables` for persistance:
```
sudo apt install iptables-persistent
sudo sh -c "iptables-save > /etc/iptables/rules.v4"
```
6. Restart services: 
```
sudo systemctl restart hostapd
sudo systemctl restart dnsmasq
```

Et Voila..
You can restart the Raspberry Pi and a hostname will appear with internet access.


   

