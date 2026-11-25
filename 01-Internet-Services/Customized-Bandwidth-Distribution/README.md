# Customized Bandwidth Distribution

## Objective
MikroTik RouterOS (hEX / RB750Gr3, RouterOS v7.12.1) ব্যবহার করে ডেডিকেটেড IP লাইনে ইন্টারনেট চালু করা এবং LAN (`192.168.88.0/24`) নেটওয়ার্কে Simple Queue দিয়ে ব্যান্ডউইথ লিমিট (`5M` Upload / `8M` Download) সেট করা।

## Lab Setup Summary
- **Router:** MikroTik hEX (RB750Gr3), RouterOS v7.12.1
- **WAN Port (`ether1`):** Dedicated Static IP (`182.163.117.146/27`), Gateway (`182.163.117.129`)
- **LAN Bridge (`bridge-lan`):** `ether2`, `ether3`, `ether4`, `ether5` (`192.168.88.1/24`)
- **DHCP Server:** `dhcp1` (`192.168.88.2-192.168.88.254`)
- **DNS:** `8.8.8.8`, `1.1.1.1`
- **NAT:** `chain=srcnat`, `out-interface=ether1`, `action=src-nat`, `to-addresses=182.163.117.146`
- **Bandwidth Queue:** `LAN-Custom-BW` on `192.168.88.0/24` with `5M` upload and `8M` download

## Step-by-Step Steps
1. Default Configuration রিমুভ করে ক্লিন স্টার্ট নেওয়া।
2. `bridge-lan` তৈরি করে `ether2`–`ether5` পোর্ট যুক্ত করা।
3. `bridge-lan`-এ `192.168.88.1/24` বসিয়ে DHCP Server (`dhcp1`) চালু করা।
4. `ether1`-এ WAN IP এবং `0.0.0.0/0` Default Route (`182.163.117.129`) সেট করা।
5. DNS (`8.8.8.8`, `1.1.1.1`) ও `src-nat` রুল তৈরি করা।
6. New Terminal থেকে `ping 8.8.8.8 count=3` দিয়ে ইন্টারনেট যাচাই করা (`0% packet loss`)।
7. `Queues -> Simple Queues`-এ `LAN-Custom-BW` (`5M` Up / `8M` Down) তৈরি করা।
