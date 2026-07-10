# FortiGate 60F — SSL VPN Installation and Setup

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0 |
| Purpose | Remote access to the home LAN via FortiClient |
| License | Not required |
| Est. time | 20–30 minutes |

## Overview

This guide configures an SSL VPN on a FortiGate 60F so a remote device running FortiClient (or a browser) can securely reach the home LAN. Split tunneling is used so the client keeps its own internet while connected.

> Replace `~~` with your site number throughout. Example: site 61 gives LAN `10.61.61.0/24` and WAN `200.0.0.61`.
>
> Screenshots are hosted on GitHub (user-attachments) and embedded inline below — no `images/` folder needed. Committing this single file is enough.

## Prerequisites

- FortiGate 60F reachable on the GUI (`https://10.~~.~~.1`)
- Admin username and password
- FortiClient installed on the phone or laptop that will connect
- The client device on a network that can reach the FortiGate WAN (`200.0.0.~~`)

## Network overview

| Item | Value |
|---|---|
| LAN interface | internal1 |
| LAN subnet | 10.~~.~~.0/24 |
| WAN interface | wan1 |
| WAN IP | 200.0.0.~~ |
| SSL VPN port | 10443 |
| Tunnel client pool | SSLVPN_TUNNEL_ADDR1 (default) |

## Configuration

### Step 1 — Create the VPN user

1. Go to User & Authentication > User Definition.
2. Click Create New.
3. Choose Local User, then Next.
4. Set a Username and Password.
5. Click through the remaining screens and Submit.

![Step 1a — User creation wizard: choose Local User](https://github.com/user-attachments/assets/b8925eae-f3ef-4818-9eaf-6095ee109537)

![Step 1b — Login Credentials: set username and password](https://github.com/user-attachments/assets/9649074a-1aec-4812-83f6-6510ea0b9305)

![Step 1c — Contact Info (optional, click through)](https://github.com/user-attachments/assets/21a11b87-5024-496e-b226-66db815da2c5)

![Step 1d — Extra Info (optional, click through)](https://github.com/user-attachments/assets/93beb0ce-c5b3-45a6-9a27-8ea2c7529cf5)

### Step 2 — Create a user group

1. Go to User & Authentication > User Groups.
2. Click Create New.
3. Name: `SSLVPN-Users`.
4. Type: Firewall.
5. Under Members, add the user from Step 1.
6. Click OK.

![Step 2 — New User Group SSLVPN-Users (Firewall) with the user added](https://github.com/user-attachments/assets/41d27125-e488-4af2-861e-465727af9fce)

### Step 3 — Create the LAN address object

1. Go to Policy & Objects > Addresses.
2. Click Create New > Address.
3. Name: `LAN-~~`.
4. Type: Subnet.
5. Subnet/IP Range: `10.~~.~~.0/24`.
6. Interface: any.
7. Click OK.

> Set Interface to `any`. Binding it to internal1 hides it from some selection menus.

![Step 3a — Policy & Objects > Addresses: Create New > Address](https://github.com/user-attachments/assets/3ebdc109-5b44-4492-b2a8-346a59424cf3)

![Step 3b — Edit Address: LAN subnet object, Interface set to any](https://github.com/user-attachments/assets/8b065ba8-55b6-4de7-b991-3a71b9c8aaac)

### Step 4 — Configure the SSL VPN portal

1. Go to VPN > SSL-VPN Portals.
2. Edit the `full-access` portal.
3. Enable Tunnel Mode.
4. For Split Tunneling, choose **Enabled Based on Policy Destinations**. The dropdown has three options: Disabled, Enabled Based on Policy Destinations, and Enabled for Trusted Destinations.
5. Click OK.

> Enabled Based on Policy Destinations builds the tunnel routes automatically from the destination of the SSL-VPN firewall policy (Step 6). With that policy's destination set to `LAN-~~`, only `10.~~.~~.0/24` goes through the tunnel and the client keeps its own internet.
> This mode requires the policy destination to be a specific subnet, not `all`, or it fails with "could not enable split tunneling as policy has all." Set the policy destination to `LAN-~~` first (Step 6), then enable this mode.
> Note: "Enabled for Trusted Destinations" (manual Routing Address Override) is meant to do the same thing, but it did not reliably push the split routes to the mobile FortiClient in this setup — the phone full-tunneled and lost internet. Policy Destinations worked.

![Step 4 — SSL-VPN Portal full-access: Tunnel Mode on, Enabled Based on Policy Destination](https://github.com/user-attachments/assets/979334b6-a882-47b2-90b4-407e2d3e1915)

### Step 5 — Configure SSL VPN settings

1. Go to VPN > SSL-VPN Settings.
2. Listen on Interface: wan1.
3. Listen on Port: `10443` (443 is the admin GUI, keep them separate).
4. Server Certificate: Fortinet_Factory (self-signed; the browser will warn, which is expected).
5. DNS Server: set to **Same as client system DNS** so the FortiGate does not push an unreachable resolver. (Blank can still push the FortiGate's own system DNS, which clients often cannot reach — that shows up as "Primary DNS is unreachable" and no internet.)
6. Under Authentication/Portal Mapping, click Create New and map `SSLVPN-Users` to the `full-access` portal.
7. Set the **All Other Users/Groups** row to a portal (e.g. `web-access`). It cannot be blank, or Apply fails with "please set the default portal for all other users/groups."
8. Click Apply.

![Step 5a — SSL-VPN Settings: listen on wan1, port 10443, Fortinet_Factory certificate](https://github.com/user-attachments/assets/5b6b5d71-230d-491c-b0a9-2f7d2d619b5f)

![Step 5b — SSL-VPN Settings: DNS "Same as client system DNS" and portal mapping (SSLVPN-Users to full-access, all others to web-access)](https://github.com/user-attachments/assets/3125e5c0-d1ed-48fe-98fc-2c8495893c8d)

### Step 6 — Create the firewall policy

1. Go to Policy & Objects > Firewall Policy.
2. Click Create New.
3. Name: `SSLVPN-to-LAN`.
4. Incoming Interface: SSL-VPN tunnel interface (ssl.root).
5. Outgoing Interface: internal1.
6. Source: `all`, and add the user group `SSLVPN-Users`.
7. Destination: `LAN-~~` (not `all`).
8. Schedule: always. Service: ALL. Action: Accept.
9. Set NAT to ON.
10. Click OK.

> NAT ON source-NATs the client to `10.~~.~~.1`, giving LAN hosts a valid return path back to the VPN client.
>
> The Destination here (`LAN-~~`) does double duty: in Split Tunneling "Based on Policy Destinations" mode, it is also the route pushed to the client. Keep it as `LAN-~~`, never `all`, or the client full-tunnels and loses internet.

![Step 6a — Firewall Policy list (Interface Pair View) before adding the SSL-VPN policy](https://github.com/user-attachments/assets/46e6d041-7622-45d2-b75c-9f29e9a0f93a)

![Step 6b — New Policy SSLVPN-TO-LAN: ssl.root to internal1, source all + SSLVPN-Users, destination LAN, NAT on](https://github.com/user-attachments/assets/c7f6a5f3-e8b0-4589-910c-a284e0c1863d)

![Step 6c — The SSLVPN-TO-LAN policy under ssl.root to internal1 in the list](https://github.com/user-attachments/assets/b3a9d02c-d291-4fb4-a9be-f726a54c0104)

### Step 7 — Connect from FortiClient

1. Open FortiClient and add a new SSL-VPN connection.

| Field | Value |
|---|---|
| Type | SSL-VPN |
| Server | 200.0.0.~~ |
| Port | 10443 |
| Username | your user |

2. Save, then Connect.
3. Accept the certificate warning.
4. Enter the password.

<img alt="Step 7 — FortiClient (iOS) SSL-VPN connection: server 200.0.0.~~:10443, port 10443" src="https://github.com/user-attachments/assets/33e0939e-03e4-48b1-a83b-438a08019936" width="340" />

> The client must be able to reach `200.0.0.~~:10443`. Quick check: open `https://200.0.0.~~:10443` in a browser first. If the login page loads, FortiClient will connect.
>
> Force-quit and reconnect FortiClient after any portal change. Routes and DNS load only at connect time.

## Verification

1. With the VPN connected, ping `10.~~.~~.1` (FortiGate LAN). It should reply.
2. Ping or open a LAN device (e.g. `https://10.~~.~~.1`). Reaching the LAN confirms the VPN works.
3. Confirm the client still browses the internet (proves split tunneling is active).

## Troubleshooting

| Symptom (VPN on) | Cause | Fix |
|---|---|---|
| Internet dies completely | Full tunnel (split routes not applied) | Set Split Tunneling to Enabled Based on Policy Destinations, and set the SSLVPN-to-LAN policy destination to `LAN-~~`. Trusted Destinations may silently full-tunnel on mobile clients |
| Sites open by IP but not by name, or "Primary DNS is unreachable" | FortiGate pushes an unreachable DNS server | Set DNS Server to "Same as client system DNS" in SSL-VPN Settings, then reconnect |
| Cannot reach LAN, gateway pings | No return path | Set NAT to ON on the ssl.root to internal1 policy |
| Cannot ping `10.~~.~~.1` at all | Wrong portal mapped or tunnel mode off | Map user to full-access, enable Tunnel Mode |
| "Could not enable split tunneling as policy has all" | Enabled Based on Policy Destinations with a policy whose destination is `all` | Set the SSLVPN-to-LAN policy destination to `LAN-~~` (not `all`), then enable Based on Policy Destinations |
| Reaches `10.~~.~~.1` but not the internet; SSLVPN-to-LAN policy shows 0 bytes | Client is full-tunneling; reaching the FortiGate LAN IP is local traffic and does not use the policy | Fix split tunneling as above (Policy Destinations + policy destination `LAN-~~`) |

## Notes

In a homelab with no real internet behind wan1, the SSL VPN reaches the LAN only. The client's internet comes from its own connection, not the FortiGate. Full tunnel with no upstream will cut the client's internet.

Split Tunneling mode matters on mobile clients. "Based on Policy Destinations" (with the SSLVPN-to-LAN policy destination set to `LAN-~~`) reliably pushed the correct split route to the phone. "For Trusted Destinations" was configured the same way but did not take effect on the mobile FortiClient, so the phone full-tunneled and lost internet. If internet drops the moment the VPN connects, this mode setting is the first thing to check.
