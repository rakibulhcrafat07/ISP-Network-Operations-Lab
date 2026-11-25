# Customized Bandwidth Distribution

## 1. Objective & Lab Overview
This lab configures a MikroTik RouterOS gateway (**hEX / RB750Gr3**, RouterOS v7.12.1) on a dedicated upstream link and applies a customized bandwidth limit (`5M` Upload / `8M` Download) across the local `192.168.88.0/24` LAN subnet using a **Simple Queue**.

- **Upstream / WAN Profile:** Dedicated static IP (`182.163.117.146/27`, Gateway `182.163.117.129`) with a day/night upstream profile (`10 Mbps` Day / `20 Mbps` Night).
- **LAN Segment:** `bridge-lan` (`192.168.88.1/24`), DHCP server `dhcp1` (`192.168.88.2-192.168.88.254`).
- **NAT Method:** Source NAT (`chain=srcnat`, `action=src-nat`, `to-addresses=182.163.117.146`, `out-interface=ether1`).
- **Bandwidth Queue:** `LAN-Custom-BW` (`target=192.168.88.0/24`, `Upload Max Limit=5M`, `Download Max Limit=8M`).

---

## 2. Topology & Port Mapping

| Component / Interface | Role | IP / Configuration | Notes |
| :--- | :--- | :--- | :--- |
| `ether1` | WAN / Upstream | `182.163.117.146/27` (GW `182.163.117.129`) | Dedicated IP, kept outside the LAN bridge |
| `bridge-lan` | LAN Bridge | `192.168.88.1/24` | Groups `ether2`–`ether5` |
| `ether2` | LAN Port | Member of `bridge-lan` | Test PC connected (`designated port`) |
| `ether3`–`ether5` | LAN Ports | Member of `bridge-lan` | Additional LAN / switch ports |
| `dhcp1` | DHCP Server | Pool `192.168.88.2-192.168.88.254` | Assigns IP addresses on `bridge-lan` |

---

## 3. Step-by-Step Configuration Guide

### Step 1: Clean Start (Remove Default Configuration)
Connect to the router via WinBox using its MAC address and remove the default configuration so pre-existing rules or IPs do not conflict with the lab setup.

- **Verification:** The default configuration prompt is removed for a clean setup.

![Remove Default Configuration](screenshots/01.%20Remove%20Configuration.png)

---

### Step 2: Create Bridge (`bridge-lan`) & Add Ports
1. From the left menu, go to **Bridge -> Bridge** and click **New (+)**.
2. Enter **Name** as `bridge-lan` and click **OK**.
3. Open the **Ports** tab and click **New (+)** to add `ether2`, `ether3`, `ether4`, and `ether5` to `bridge-lan`.

- **Verification:** `bridge-lan` is created and `ether2` through `ether5` appear under **Bridge -> Ports** with `ether2` active (`H`, `designated port`).

![LAN Bridge Window](screenshots/02.%20Lan%20Bridge%20Create%201.png)
![Create bridge-lan](screenshots/02.%20Lan%20Bridge%20Create%202.png)
![Add Bridge Ports](screenshots/02.%20Lan%20Bridge%20Create%203.png)
![Bridge Ports Active](screenshots/02.%20Lan%20Bridge%20Create%204.png)

---

### Step 3: Assign LAN IP, DHCP Server, WAN IP, DNS & Default Route
1. Assign `192.168.88.1/24` on `bridge-lan` and configure DHCP server `dhcp1` (`192.168.88.2-192.168.88.254`).
2. Assign `182.163.117.146/27` on `ether1`, add default route `0.0.0.0/0` via `182.163.117.129`, and configure DNS servers (`8.8.8.8`, `1.1.1.1`).

---

### Step 4: Configure Source NAT (`src-nat`)
Because this lab uses a dedicated static IP, outbound LAN traffic is translated using `src-nat` to `182.163.117.146`.

1. Go to **IP -> Firewall -> NAT** and click **New (+)**.
2. On the **General** tab, set **Chain** to `srcnat` and **Out. Interface** to `ether1`.
3. On the **Action** tab, set **Action** to `src-nat` and **To Addresses** to `182.163.117.146`.
4. Click **Apply** and then **OK**.

- **Verification:** The `srcnat` rule (`action=src-nat`, `out-interface=ether1`) is saved in **Firewall -> NAT**.

![Firewall NAT src-nat Rule](screenshots/04.%20NAT%207.png)

---

### Step 5: Verify Internet Connectivity (CLI Ping)
1. Open **New Terminal** from the left menu in WinBox.
2. Run `ping 8.8.8.8 count=3` to check external reachability.

- **Verification:** Replies succeed with `sent=3 received=3 packet-loss=0%` and average RTT around `31ms`–`32ms`.

![CLI Reachability Ping Verification](screenshots/05.%20Reachability.png)

---

### Step 6: Configure Customized Bandwidth Queue (`Simple Queue`)
1. Go to **Queues -> Simple Queues**.
2. Open the queue rule (or click **New (+)**) and configure on the **General** tab:
   - **Name:** `LAN-Custom-BW`
   - **Target:** `192.168.88.0/24`
   - **Target Upload (Max Limit):** `5M`
   - **Target Download (Max Limit):** `8M`
3. Click **Apply** and then **OK**.

- **Verification:** `LAN-Custom-BW` (`target=192.168.88.0/24`) is active in **Simple Queues** with `Upload Max Limit = 5M` and `Download Max Limit = 8M`.

![Simple Queues Initial](screenshots/06.%20Queue%201.png)
![Simple Queue Edit Window](screenshots/06.%20Queue%202.png)
![Simple Queue Configured](screenshots/06.%20Queue%203.png)

---

## 4. Notes & Lessons Learned
- `LAN-Custom-BW` shapes the combined (shared) bandwidth of the entire `192.168.88.0/24` LAN subnet to `5M` upload and `8M` download.
- Any additional screenshots uploaded later into `screenshots/` can be linked directly in `README.md`.
