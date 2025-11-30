# Configuration Guide: OPNsense + NordVPN (WireGuard)

This document provides a comprehensive guide on configuring a WireGuard (NordLynx) interface in OPNsense using a NordVPN subscription, including steps to obtain necessary credentials and configure the connection.

[Traducción en español](README_es.md)

Versions:
*   OPNsense 25
*   NordVPN Active Subscription (2026)


## 1. Get NordVPN configuration

Before configuring OPNsense, you need to obtain your credentials and keys via a terminal.

### Step 1.1: Obtain your Private Key
For some reason, NordVPN hides this; it is possible they only want us to use their own applications.

Navigate to "Access Tokens" and create a new one with permissions to "Get service credentials".
1.  [NordAccount](https://my.nordaccount.com/dashboard/) >
2.  [NordVPN](https://my.nordaccount.com/dashboard/nordvpn/) >
3.  [Access token](https://my.nordaccount.com/dashboard/nordvpn/access-tokens/) >
4.  You will need to verify your email address.
5.  Create a new token with the "Get service credentials" permission. Let's assume the generated token is `e9f2___X___66`.
6.  The next step is to use that token to obtain your WireGuard Private Key.

```bash
curl -s -u token:e9f2___X___66 https://api.nordvpn.com/v1/users/services/credentials | jq -r .nordlynx_private_key
Vnn______________fMQ=
```

**Note:** You must have `jq` installed to process JSON. https://jqlang.org/download/

The command above returns your WireGuard **Private Key**.


### Step 1.2: Obtain Service Credentials

Navigate to the **Manual Setup** section.
1.  [NordAccount](https://my.nordaccount.com/dashboard/) >
2.  [NordVPN](https://my.nordaccount.com/dashboard/nordvpn/) >
3.  [Manual Setup](https://my.nordaccount.com/dashboard/nordvpn/manual-configuration/) >
4.  [Tab "Service Credentials"](https://my.nordaccount.com/dashboard/nordvpn/manual-configuration/service-credentials/) >
5.  "Verify your email" is required.
6.  Copy your service **Username** and **Password** (long strings, not your email).


### Step 1.3: Obtain Server Data
Run this to obtain the IP and Public Key of the best server, in my case in Spain (ID 69):

```bash
curl -s "https://api.nordvpn.com/v1/servers/recommendations?filters\[country_id\]=202&limit=1" | jq -r '.[0] | {hostname: .hostname, ip: .station, public_key: (.technologies[] | select(.identifier=="wireguard_udp") | .metadata[] | select(.name=="public_key") | .value)}'

{
  "hostname": "es225.nordvpn.com",
  "ip": "185.214.97.110",
  "public_key": "OaSGa___________AXY="
}
```


## 2. WireGuard Configuration in OPNsense

To create a WireGuard connection, it is necessary to create an "Instance" (your connection) and a "Peer" (the remote server).

### Step 2.1: Configure your Connection (Instance)
Go to **VPN > WireGuard > Instances** and add a new one:

*   **Name:** `NordVPN_Local`
*   **Private Key:** The key `Vnn______________fMQ=` that you obtained in step 1.1.
*   **Public Key:** Leave empty; it calculates automatically.
*   **Listen Port:** `51820`
*   **DNS Server:** `103.86.96.100` (NordVPN DNS).
*   **Tunnel Address:** `10.5.0.2/32` (Internal IP to be used as NordLynx Gateway).
*   **Disable Routes:** **CHECKED ✅** In my case, I do not want all router traffic to pass through the VPN, only that of a specific VLAN. If you want ALL router traffic to pass through the VPN, leave unchecked.
*   **Save**
*   **Apply**


### Step 2.2: Configure the Remote Server (Peer)
Go to **VPN > WireGuard > Peers** and add a new one:

*   **Name:** `NordVPN_ES_225` (In this case, I want to identify the Spain server 225).
*   **Public Key:** The server's `public_key` obtained in step 1C (e.g., `OaSGa___________AXY=`).
*   **Endpoint Address:** The numeric `ip` of the server obtained in step 1C (e.g., `185.214.9.110`).
*   **Endpoint Port:** `51820`
*   **Allowed IPs:** `0.0.0.0/0` (Allow all internet traffic).
*   **Keepalive Interval:** `25` (Set persistent keepalive interval in seconds).
*   **Save**
*   **Apply**

### Step 2.3: Link Peer to Instance
Go to **VPN > WireGuard > Instances** and link the remote server to your connection:

*  Edit the `NordVPN_Local` instance.
*  In the **Peers** tab, add the peer `NordVPN_ES_225`.
*   **Save**
*   **Apply**

### Step 2.4: Enable the Peer
Go to **VPN > WireGuard > Peers**:

* Edit the peer `NordVPN_ES_225`, click the pencil icon.
* **Enable:** Check ✅ This will enable the peer.
*   **Save**
*   **Apply**

### Step 2.5: Verify Connection
Go to **VPN > WireGuard > Status**:

It must show the interface and the peer active. Both with a green check ✅.

OPNsense should have created the `wg0` interface automatically; the interface and the peer use this interface. If everything went well, the peer shows TX/RX traffic and the duration since the last "Handshake".

In case of issues, check the logs in **VPN > WireGuard > Log File**.
