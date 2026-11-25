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
Connect to the router via WinBox using its MAC address and remove the default configuration so no pre-existing firewall rules, routes, or IP addresses conflict with the lab setup.

- **Verification:** The router reconnects cleanly in WinBox after removing the default configuration.

---

### Step 2: Create Bridge (`bridge-lan`) & Add Ports
1. From the left menu, go to **Bridge -> Bridge** and click **New (+)**.
2. Enter **Name** as `bridge-lan` and click **OK**.
3. Open the **Ports** tab and click **New (+)** to add `ether2`, `ether3`, `ether4`, and `ether5` to `bridge-lan`.

- **Verification:** In the session captures, `bridge-lan` was created and `ether2` through `ether5` appeared under **Bridge -> Ports** with `ether2` active (`H`, `designated port`) and `ether3`–`ether5` showing `IH` (`disabled port`) while unlinked.

---

### Step 3: Assign LAN IP & Configure DHCP Server
1. Go to **IP -> Addresses** and click **New (+)**.
2. Set **Address** to `192.168.88.1/24` and **Interface** to `bridge-lan`, then click **OK**.
3. Go to **IP -> DHCP Server** and click **DHCP Setup**.
4. Select `bridge-lan` as the **DHCP Server Interface** and click **Next** through the gateway (`192.168.88.1`), address pool, and DNS prompts until setup finishes (`dhcp1`).

- **Verification:** In the session captures, `192.168.88.1/24` was added on `bridge-lan` (`192.168.88.0` network), and **DHCP Setup** completed with `dhcp1` (`bridge-lan`, lease `00:30:00`, pool `dhcp_pool1`).

---

### Step 4: Configure WAN IP & DNS
1. Go to **IP -> Addresses** and click **New (+)**.
2. Set **Address** to `182.163.117.146/27` and **Interface** to `ether1`, then click **OK**.
3. Open **IP -> DNS**, enter `8.8.8.8` and `1.1.1.1` under **Servers**, and click **OK**.

- **Verification:** In the session captures, `182.163.117.146/27` was entered on `ether1` in **Address -> New**, and `8.8.8.8` and `1.1.1.1` were configured in **DNS Settings**.

---

### Step 5: Add Default Route (`0.0.0.0/0`)
1. Go to **IP -> Routes** and click **New (+)**.
2. Set **Dst. Address** to `0.0.0.0/0` and **Gateway** to `182.163.117.129`.
3. Click **Apply** and then **OK**.

- **Verification:** In the session captures, **Route List** initially showed connected routes (`DAC`) for `182.163.117.128/27` (`ether1`) and `192.168.88.0/24` (`bridge-lan`), and after adding `0.0.0.0/0` via `182.163.117.129`, an `AS` route with `Distance = 1` appeared.

---

### Step 6: Configure Source NAT (`src-nat`)
Because this lab uses a dedicated static IP, outbound traffic from the LAN is translated using `src-nat` to `182.163.117.146`.

1. Go to **IP -> Firewall -> NAT** and click **New (+)**.
2. On the **General** tab, set **Chain** to `srcnat` and **Out. Interface** to `ether1`.
3. On the **Action** tab, set **Action** to `src-nat` and **To Addresses** to `182.163.117.146`.
4. Click **Apply** and then **OK**.

- **Verification:** In the session captures, the NAT rule appeared in **Firewall -> NAT** with `Action = src-nat`, `Chain = srcnat`, and `Out. Interface = ether1`.

---

### Step 7: Verify Internet Connectivity (CLI Ping)
1. Open **New Terminal** from the left menu in WinBox.
2. Run the following command:
   ```routeros
   ping 8.8.8.8 count=3

   ### Step 8: Configure Customized Bandwidth Queue (`Simple Queue`)
1. Go to **Queues -> Simple Queues**.
2. Open the queue rule (or click **New (+)**) and configure the **General** tab:
   - **Name:** `LAN-Custom-BW`
   - **Target:** `192.168.88.0/24`
   - **Target Upload (Max Limit):** `5M`
   - **Target Download (Max Limit):** `8M`
3. Click **Apply** and then **OK**.

- **Verification:** In the session captures, `LAN-Custom-BW` (`target=192.168.88.0/24`) was saved in **Simple Queues** with `Upload Max Limit = 5M` and `Download Max Limit = 8M`.

## 4. Notes & Lessons Learned
- `LAN-Custom-BW` shapes the combined (shared) bandwidth of the entire `192.168.88.0/24` LAN subnet to `5M` upload and `8M` download.
- If new screenshots are captured later, place them in a `screenshots/` folder inside this lab directory and link them in this `README.md`.
