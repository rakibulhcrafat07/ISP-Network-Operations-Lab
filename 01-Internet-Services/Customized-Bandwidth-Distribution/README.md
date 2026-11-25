# Customized Bandwidth Distribution

## 1. Objective & Lab Overview
This lab documents a comprehensive, production-minded MikroTik RouterOS gateway configuration on a **MikroTik hEX (RB750Gr3)** running **RouterOS v7.12.1**, starting cleanly from a wiped baseline without any default configuration rules or pre-existing address conflicts. The goal is to build a structured, repeatable network setup where `ether1` operates as a dedicated static WAN interface (`182.163.117.146/27` with default gateway `182.163.117.129` against a `10 Mbps` Day / `20 Mbps` Night upstream profile), while local interfaces `ether2` through `ether5` are grouped inside a Layer 2 software bridge (`bridge-lan`) assigned IPv4 gateway `192.168.88.1/24` with an active DHCP server (`dhcp1`, lease pool `192.168.88.2-192.168.88.254`) and public DNS resolvers (`8.8.8.8` and `1.1.1.1`). Outbound LAN-to-WAN traffic is translated onto the dedicated public IPv4 address using explicit Source NAT (`src-nat`), and total subnet throughput is shaped with a RouterOS **Simple Queue** (`LAN-Custom-BW`) enforcing a `5M` upload ceiling and `8M` download ceiling across `192.168.88.0/24`.

- **Upstream / WAN Profile:** Dedicated static IPv4 address `182.163.117.146/27` configured on `ether1` with default gateway `182.163.117.129`, tested against an upstream service profile providing `10 Mbps` during daytime hours and `20 Mbps` during nighttime hours so behavior under constrained bandwidth can be observed clearly.
- **LAN Segment & DHCP Service:** Layer 2 bridge interface `bridge-lan` combining Ethernet ports `ether2`, `ether3`, `ether4`, and `ether5` under a single local broadcast domain with gateway IP `192.168.88.1/24`, paired with DHCP server `dhcp1` distributing dynamic IP leases from `192.168.88.2` to `192.168.88.254` along with subnet mask, gateway, and DNS settings (`8.8.8.8`, `1.1.1.1`).
- **Network Address Translation (NAT):** Dedicated Source NAT rule in the `srcnat` chain (`chain=srcnat`, `action=src-nat`, `to-addresses=182.163.117.146`, `out-interface=ether1`) translating private RFC1918 source addresses from `192.168.88.0/24` into the router's assigned dedicated static WAN IP while maintaining session state in the RouterOS connection tracking table.
- **Bandwidth & Traffic Shaping Policy:** RouterOS Simple Queue rule named `LAN-Custom-BW` targeting `192.168.88.0/24` with **Upload Max Limit** configured to `5M` (`5 Mbps`) and **Download Max Limit** configured to `8M` (`8 Mbps`) to regulate shared LAN bandwidth consumption at the edge.

---

## 2. Topology & Port Mapping

| Component / Interface | Role | IP / Configuration | Notes |
| :--- | :--- | :--- | :--- |
| `ether1` | WAN / Upstream Interface | `182.163.117.146/27` (Gateway `182.163.117.129`) | Excluded from `bridge-lan`; handles default route `0.0.0.0/0`, upstream reachability, and outbound `src-nat` translation |
| `bridge-lan` | Layer 2 LAN Bridge | `192.168.88.1/24` | Software bridge joining `ether2`–`ether5` into one switched LAN domain with gateway IP `192.168.88.1` |
| `ether2` | LAN Access Port | Member of `bridge-lan` | Connected to test client (`H`, `designated port`, Path Cost `10`, PVID `1`) for connectivity and rate-limit testing |
| `ether3`–`ether5` | LAN Switch Ports | Members of `bridge-lan` | Bridged local Ethernet ports (`PVID 1`, `Path Cost 10`) available for additional PCs, access points, or switches |
| `dhcp1` | IPv4 DHCP Server | Pool `192.168.88.2-192.168.88.254` | Automatically leases IPv4 addresses, `/24` netmask, default gateway `192.168.88.1`, and DNS servers to LAN hosts |

---

## 3. Step-by-Step Configuration Guide

### Step 1: Clean Start (Remove Default Configuration)
When beginning a custom ISP gateway and traffic-shaping lab on a **MikroTik hEX (RB750Gr3)** running **RouterOS v7.12.1**, removing the factory default configuration is an important first step. RouterOS default scripts typically pre-populate a default `192.168.88.1/24` bridge, a DHCP client on `ether1`, masquerade NAT rules, and default firewall filter entries. Removing those defaults ensures every interface, IP subnet, route, NAT rule, and queue policy in the lab is created intentionally and cleanly without hidden side effects or conflicting rules.

1. Launch **WinBox** on your management workstation, discover the MikroTik hEX (RB750Gr3) over Layer 2 using its MAC address if needed, and connect to the router.
2. When the **RouterOS Default Configuration** dialog appears on screen, select **Remove Configuration** so the router clears default scripts and boots into a clean baseline state.
3. Verify that no leftover default bridge assignments, conflicting IP addresses, or unwanted firewall/NAT entries remain before configuring custom LAN and WAN parameters.

- **Verification:** The default configuration prompt is removed cleanly, leaving the MikroTik hEX (RB750Gr3) in a clean state ready for step-by-step bridge, IP, routing, NAT, and Simple Queue configuration.

![Remove Default Configuration](screenshots/01.%20Remove%20Configuration.png)

---

### Step 2: Create Bridge (`bridge-lan`) & Add Ports
In MikroTik RouterOS, creating a bridge interface (`bridge-lan`) allows multiple physical Ethernet ports (`ether2` through `ether5`) to operate together as a single switched Layer 2 LAN segment while keeping `ether1` separate as the dedicated WAN port. Attaching the Layer 3 LAN gateway IP (`192.168.88.1/24`) and DHCP server (`dhcp1`) to `bridge-lan`—rather than to a single Ethernet port—ensures any host plugged into `ether2`–`ether5` shares the same local subnet and gateway.

1. In the left navigation pane of WinBox, click **Bridge** to open the **Bridge** management window, stay on the **Bridge** tab, and click **New (+)** to create a new bridge interface.
2. On the **General** tab of the new interface dialog, set **Name** to `bridge-lan` (confirming **Enabled** is checked and **Fast Forward** remains enabled for normal forwarding performance), then click **Apply** and **OK**.
3. Switch to the **Ports** tab inside the **Bridge** window and click **New (+)** repeatedly to add `ether2`, `ether3`, `ether4`, and `ether5` as bridge ports with **Bridge** set to `bridge-lan`.
4. Review the **Bridge -> Ports** table to verify that `ether2`–`ether5` are listed under `bridge-lan` with `PVID = 1` and `Path Cost = 10`, noting that connected active ports display flags such as `H` (`hardware offload`) and role `designated port`, whereas unconnected ports show `IH` (`inactive`) and `disabled port` until a link comes up.

- **Verification:** `bridge-lan` is created and active, and all four LAN ports (`ether2`, `ether3`, `ether4`, and `ether5`) are properly attached to `bridge-lan` under **Bridge -> Ports**.

![LAN Bridge Window](screenshots/02.%20Lan%20Bridge%20Create%201.png)
![Create bridge-lan](screenshots/02.%20Lan%20Bridge%20Create%202.png)
![Add Bridge Ports](screenshots/02.%20Lan%20Bridge%20Create%203.png)
![Bridge Ports Active](screenshots/02.%20Lan%20Bridge%20Create%204.png)

---

### Step 3: Assign LAN IP, DHCP Server, WAN IP, DNS & Default Route
Once `bridge-lan` groups `ether2`–`ether5` at Layer 2 and `ether1` remains isolated for upstream access, Layer 3 addressing, local IP lease assignment, DNS resolution, and default routing must be configured so LAN clients can reach the router and the router can forward traffic to the ISP upstream gateway.

1. **Configure LAN Gateway & DHCP Server (`dhcp1`):** Navigate to **IP -> Addresses**, add static IPv4 address `192.168.88.1/24` on interface `bridge-lan`, and then run **IP -> DHCP Server -> DHCP Setup** on `bridge-lan` to create server `dhcp1` with address pool `192.168.88.2-192.168.88.254`, gateway `192.168.88.1`, and DNS servers so local workstations obtain IP addresses automatically.
2. **Configure Dedicated WAN IP (`ether1`), Default Route & DNS:** Under **IP -> Addresses**, assign the dedicated static WAN address `182.163.117.146/27` to `ether1` (outside `bridge-lan`). Under **IP -> Routes**, add a static default route with **Dst. Address** `0.0.0.0/0` and **Gateway** `182.163.117.129` so traffic destined for external networks is forwarded to the upstream ISP gateway. Under **IP -> DNS**, configure servers `8.8.8.8` and `1.1.1.1` (and enable **Allow Remote Requests** if LAN clients query the router's IP for DNS).

- **Verification:** Clients connected to `bridge-lan` receive an IP address from `192.168.88.2-192.168.88.254` with gateway `192.168.88.1`, `ether1` holds `182.163.117.146/27`, and the routing table shows a valid active default route (`0.0.0.0/0`) via gateway `182.163.117.129`.

---

### Step 4: Configure Source NAT (`src-nat`)
Because local endpoints on `bridge-lan` use private RFC1918 addresses (`192.168.88.0/24`) that are not routed across the public internet, outbound packets leaving `ether1` toward the upstream ISP must undergo Network Address Translation. Configuring an explicit Source NAT (`src-nat`) rule translates the packet source IP address to the dedicated static WAN address (`182.163.117.146`) and records session mappings in RouterOS connection tracking so return traffic is routed back to the originating LAN host.

1. In WinBox, open **IP -> Firewall** from the left menu, select the **NAT** tab, and click **New (+)** to create a new NAT rule.
2. On the **General** tab, set **Chain** to `srcnat` (which processes packets right before they exit the router) and set **Out. Interface** to `ether1` so translation applies specifically to traffic leaving through the WAN interface.
3. Switch to the **Action** tab, set **Action** to `src-nat`, and enter `182.163.117.146` in **To Addresses** so outgoing traffic is explicitly translated using the dedicated static IPv4 address.
4. Click **Apply** to install the rule into the active RouterOS firewall configuration and click **OK** to close the window.

- **Verification:** The rule (`chain=srcnat`, `action=src-nat`, `out-interface=ether1`) is saved and active in **IP -> Firewall -> NAT**, ready to translate outbound LAN traffic and increment packet/byte counters.

![Firewall NAT src-nat Rule](screenshots/04.%20NAT%207.png)

---

### Step 5: Verify Internet Connectivity (CLI Ping)
Before tuning or validating bandwidth limits with a Simple Queue, it is best practice to verify end-to-end IP reachability, routing table selection, `src-nat` translation, and round-trip latency directly from the MikroTik RouterOS command-line interface (CLI). Testing against a well-known external IP address (`8.8.8.8`) isolates basic Layer 3 packet delivery and gateway reachability.

1. Click **New Terminal** on the left menu in WinBox to open the interactive RouterOS v7.12.1 CLI terminal connected to the MikroTik hEX (RB750Gr3).
2. Execute `ping 8.8.8.8 count=3` to transmit three ICMP Echo Request packets to `8.8.8.8` and observe the reply status, packet size, TTL, and round-trip time (RTT) across multiple consecutive command runs.

- **Verification:** Each `ping 8.8.8.8 count=3` run completes with `sent=3 received=3 packet-loss=0%` and consistent round-trip latency around `31ms`–`32ms`, confirming that `ether1`, gateway `182.163.117.129`, default route `0.0.0.0/0`, and `src-nat` translation to `182.163.117.146` are working reliably.

![CLI Reachability Ping Verification](screenshots/05.%20Reachability.png)

---

### Step 6: Configure Customized Bandwidth Queue (`Simple Queue`)
RouterOS **Simple Queues** provide a straightforward way to perform traffic shaping and bandwidth rate limiting on specific IP addresses, subnets, or interfaces. In this lab, a Simple Queue named `LAN-Custom-BW` is applied to the `192.168.88.0/24` LAN subnet to cap upload speed at `5M` (`5 Mbps`) and download speed at `8M` (`8 Mbps`) regardless of whether the upstream link is operating in its `10 Mbps` day or `20 Mbps` night profile.

1. Open **Queues** from the left menu in WinBox and stay on the **Simple Queues** tab to inspect existing queue rules, target subnets, upload/download limits, and live packet/byte counters.
2. Select the queue rule to edit (or click **New (+)** to create a new Simple Queue entry) and set the following fields on the **General** tab:
   - **Name:** `LAN-Custom-BW` (assigns a descriptive name to identify the custom local LAN bandwidth limit in the queue table)
   - **Target:** `192.168.88.0/24` (matches traffic associated with the entire `192.168.88.0/24` LAN subnet on `bridge-lan`)
   - **Target Upload (Max Limit):** `5M` (sets the maximum upload rate limit to `5 Mbps` for outbound traffic from the target)
   - **Target Download (Max Limit):** `8M` (sets the maximum download rate limit to `8 Mbps` for inbound traffic toward the target)
3. Click **Apply** to apply the rate limit immediately in RouterOS and click **OK** to save and close the queue configuration dialog.

- **Verification:** `LAN-Custom-BW` (`target=192.168.88.0/24`) appears enabled in **Queues -> Simple Queues** with **Upload Max Limit** set to `5M` and **Download Max Limit** set to `8M`, actively shaping traffic for the LAN subnet.

![Simple Queues Initial](screenshots/06.%20Queue%201.png)
![Simple Queue Edit Window](screenshots/06.%20Queue%202.png)
![Simple Queue Configured](screenshots/06.%20Queue%203.png)

---

## 4. Notes & Lessons Learned
- **Shared Subnet Rate Limiting Behavior:** A Simple Queue configured with `Target = 192.168.88.0/24` and `Max Limit = 5M / 8M` shapes the combined traffic of the entire `192.168.88.0/24` subnet to a `5M` upload ceiling and `8M` download ceiling—it does not assign a separate dedicated `5M/8M` bandwidth allocation to each individual client IP address.
- **Importance of Clean Layer 2/Layer 3 Separation & `src-nat`:** Keeping `ether1` dedicated to the static WAN IP (`182.163.117.146/27`) while bridging `ether2`–`ether5` under `bridge-lan` (`192.168.88.1/24`), configuring default routing via `182.163.117.129`, and setting up `src-nat` (`to-addresses=182.163.117.146`) ensures traffic is properly routed and translated before Simple Queue shaping takes effect.
- **GitHub Documentation & Asset Paths:** Storing all PNG screenshots inside `screenshots/` alongside `README.md` and using `%20` URL encoding for spaces in relative Markdown links keeps documentation organized and ensures images render cleanly in GitHub.
