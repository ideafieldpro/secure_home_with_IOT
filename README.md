# Secure Home Network and Home Automation System Design

## Project Overview
Design and implement a secure, segmented home network architecture combining with a home automation platform that includes strong security and accessibility. Leverage OpenWrt on a consumer-grade router for advanced networking features and Proxmox VE for virtualization hosting Home Assistant.

## Hardware and Software Used
- Consumer-grade router flashed with OpenWrt firmware for VLAN and firewall customization
- Proxmox VE server
- Home Assistant running as a VM on Proxmox for home automation control
- Network cables, adapters, and various IOT devices such as cameras, electrical switches, lights, and IR blasters

## Network Design and Security
- Configured multiple VLANs on the OpenWrt router:
  - IoT VLAN isolated from the main personal device VLAN
- Enforced strict firewall rules between VLANs to constrain lateral movement
- Segmentation reduces exposure of sensitive devices and limits attack surface
- Implemented DHCP reservations and monitored traffic using OpenWrt tools

---

## Considerations

### Router
The Cudy WR3000S was selected as the core router platform for its combination of affordability, OpenWrt compatibility, and hardware features that fit the project’s security and automation goals. Some of the primary reasons for choosing this model:
- OpenWrt Compatibility: The WR3000S is fully supported by OpenWrt v24+, making it easy to leverage advanced VLAN, firewall, and remote management features.​
- WPA3 Support: The router hardware and OpenWrt build offer robust WPA3 wireless security out-of-the-box, which was a key requirement for protecting client network traffic.​
- Ample Storage (128MB Flash): Unlike similar models with just 16MB of storage, WR3000S features 128MB NAND flash, ensuring space to install extra packages for VPN, DNS filtering, remote support, WireGuard/OpenVPN client/server, and other management tools needed for a future-proof install.​
- Reliable Performance: With a 1.3GHz quad-core CPU, gigabit ports, and Wi-Fi 6, this router sustains high device counts and throughput, including for LAN and VLAN traffic requirements.​
- Cost-effective and Compact: Delivers solid features at a lower cost than many enterprise solutions, and small size is ideal for a home setting.

  ### Points to check when selecting a router:
  - OpenWrt support: Official support for your model and revision (see [OpenWrt Table of Hardware](https://toh.openwrt.org/?view=normal))​
  - Flash and RAM: Minimum 16MB flash and 128MB RAM; more is better for packages, logs, and remote access.​
  - WPA3 capability: Secure modern Wi-Fi authentication.​
  - Gigabit ports: Needed for high-speed LAN and WAN traffic.
  - CPU Strength: Ensures router can handle multiple VLANs, firewall rules, and VPNs with low latency.
  - Package space: Enough flash space for custom OpenWrt packages (e.g. wireguard, openvpn, adblock, luci-apps).
  - Vendor openness: Avoid Broadcom chipsets which have poor open-source support.
 
### Proxmox VE Server
The Beelink N100 Mini PC (also branded as Mini S12 Pro) was chosen primarily for its affordability and sufficient CPU and RAM capabilities to run the Home Assistant VM without extra overhead. Its Intel Alder Lake N100 quad-core processor at up to 3.4GHz and support for up to 16GB RAM provide a balanced platform tailored to the project’s modest resource needs.

One limitation discovered during deployment was that this model only comes with a single built-in Gigabit Ethernet NIC. This presented a challenge because the project’s network design required multiple physical NICs to separate VLAN traffic, due to a limitation on the Cudy WR3000S ports.

The single NIC constraint was overcome by using a USB to Ethernet adapter plugged into the Beelink N100. This inexpensive USB Ethernet solution effectively added a second NIC, allowing proper VLAN bridging and network segmentation despite the hardware limitations of the mini PC.

This experience highlights the importance of NIC count for network routing and virtualization projects, and how pragmatic workarounds can adapt budget hardware to professional-grade network topologies.

  ### Server selection tips:
  - CPU Performance
     - Select a CPU with multiple cores and good single-thread performance to efficiently run Home Assistant and any lightweight supporting containers or VMs.
     - A modern Intel or AMD processor from recent generations is recommended for energy efficiency and virtualization features (e.g., VT-x/AMD-V).
  - Memory (RAM)
     - Home Assistant and additional services require at least 8GB RAM for smooth operation; 16GB offers better headroom for future expansion.
     - More RAM allows for running additional VMs/containers if needed.
  - Storage
     - Use SSD storage for the OS and VM disks to ensure fast boot and responsiveness.
     - Consider NVMe drives if budget permits for even better performance.
     - Sufficient capacity to hold backups, snapshots, and VM images is important (256GB or more recommended).
  - Network Interfaces
     - Multiple physical NICs are highly beneficial for VLAN bridging and isolating traffic per network segment, especially in segmented setups.
     - At least two NICs are recommended: one for WAN or management, another dedicated to IoT or guest VLANs.
  - Expansion Ports and Connectivity
     - USB ports can be useful for adding additional NICs (e.g., via USB to Ethernet adapters) if physical NICs are limited.
     - Consider availability of PCIe slots if you plan advanced networking cards in the future.
  - Power and Noise
     - For a home environment, low power consumption and quiet operation are desirable.
     - Mini-PCs with efficient cooling strike a good balance between performance and noise.
  - Budget and Future Proofing
     - Choose hardware that fits your budget but leaves room for future service expansions or increased automation complexity.
     - Avoid overpaying for unused capabilities but ensure the platform won't bottleneck soon.
  - Reliability and Support
     - Opt for well-supported hardware with good community or vendor support.
     - Confirm Proxmox compatibility or ensure drivers for network/storage devices are available on Debian-based systems.

### Network segmentation
The network was divided into three distinct segments to balance security, accessibility, and management efficiency:

Management VLAN: This primary segment hosts critical devices such as personal computers, servers, and administration tools. Isolating management traffic ensures only trusted devices have full network access, reducing risk from less secure endpoints.

Guest VLAN: A separate network for visitors provides internet access without exposing sensitive devices or data in the primary network, maintaining client privacy and preventing unauthorized access.

IoT VLAN: IoT devices often have weaker security and pose a higher risk of compromise. Placing them on their own isolated VLAN mitigates the threat of lateral movement and prevents potential attacks from reaching the primary network and management devices.

Segmenting in this way enhances overall security by minimizing attack surfaces, improves network stability by limiting broadcast traffic within VLANs, and simplifies monitoring and applying tailored firewall rules for each segment.

---

# Installation

## OpenWRT
Installing OpenWrt unlocks powerful networking features that consumer-grade firmware typically lacks. Below is a general guide to flashing OpenWrt on routers similar to the Cudy WR3000S.

### Step 1: Verify Router Compatibility
Confirm your router model and hardware version is supported by OpenWrt.
Check the OpenWrt Table of Hardware and search for your device to find compatible firmware images and notes.

### Step 2: Download the Correct OpenWrt Firmware
Visit the OpenWrt Firmware Selector or the OpenWrt download page for your router.
Download the correct release image for your router model and hardware revision.
For Cudy WR3000S, ensure you select images built for that specific device.

### Step 3: Prepare for Installation
Connect your computer to the router via Ethernet to avoid interruptions during the process.
Set a static IP on your PC aligned with the router’s default management subnet if necessary (e.g., 192.168.1.x).

### Step 4: Flash the Firmware
Access the router’s current administration interface (usually at http://192.168.1.1).
Locate the firmware upgrade section.
Upload the OpenWrt firmware image.
Confirm and wait patiently for the router to flash and reboot. Interrupting this process can brick the device.

### Step 5: Initial Access and Configuration
After reboot, connect to the new OpenWrt interface at http://192.168.1.1.
No password is set initially; set a strong root password immediately.
Begin configuring network settings including VLANs, Wi-Fi security (WPA3), firewall zones, and packages for remote management.

### Useful Resources
- OpenWrt Official Installation Guide: https://openwrt.org/docs/guide-quick-start/flashing
- Cudy WR3000S OpenWrt Community Discussions: https://forum.openwrt.org/t/openwrt-on-cudy-wr3000s/233708
- OpenWrt Firmware Selector: https://firmware-selector.openwrt.org/
- OpenWrt Table of Hardware (Compatibility): https://toh.openwrt.org/?view=normal

---

## Proxmox
Installing Proxmox VE on a dedicated PC turns it into a powerful virtualization platform for managing home automation, network segmentation, and other services. Here’s a step-by-step guide to get you started.

### Step 1: Prepare the Hardware
- Select a PC with a compatible processor (Intel or AMD), at least 8GB RAM, and SSD storage (NVMe or SATA).
- Ensure the PC has networking capabilities suitable for your segmentation needs.
- Optional: Add multiple NICs for network flexibility.

### Step 2: Download Proxmox VE ISO
- Visit the [Proxmox Download page](https://www.proxmox.com/en/downloads).
- Download the latest ISO installer image compatible with your hardware.

### Step 3: Create Bootable USB Installer
- Use a tool like Etcher (Windows/Mac/Linux) to create a bootable USB drive:
- Resource: [Proxmox USB Boot Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#_usb_installation)

### Step 4: BIOS/UEFI Settings for Boot
- Insert the USB installer into your PC and power on.
- Enter BIOS/UEFI settings (usually by pressing Del, F2, or F12 during boot).

Configure the following:
- Set boot priority to boot from the USB drive first.
- Disable Secure Boot if enabled.
- Enable UEFI mode if supported, or leave in Legacy mode based on your hardware.
- Enable TPM or secure boot only if required, but typically disabled for Proxmox.

Save and Exit.

### Step 5: Enable Automatic Power-On
- While in BIOS/UEFI, find settings like "Restore on AC Power Loss" or "AC Back".
- Set it to "Power On" or "Always On" so that the device automatically powers on after power loss.

Note: This setting ensures your Proxmox server will automatically start without manual intervention after power outages or restarts.

### Step 6: Install Proxmox VE
- Boot from the USB drive.
- Follow on-screen prompts:
  - Choose the target disk.
  - Set the country, timezone, and keyboard layout.
  - Define the admin password and network settings.
- Finish installation and remove the USB when prompted.
- Reboot the PC; it should now boot into Proxmox automatically.

### Step 7: Post-Installation Setup
- Log in via the web interface at https://<your-ip>:8006.
- Complete network configuration, storage setup, and VM deployment as needed.

### Resources:
- [Proxmox VE Official Installation Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#_installation)
- [Official Etcher Website](https://etcher.balena.io/)

---

# Configurations

## OpenWRT bridges, VLANs, & firewall

When placing OpenWrt behind an ISP-provided router (in a double NAT or routed mode), careful network and firewall configuration is essential to maintain segmentation and security.

### Step 1: Connect OpenWrt WAN to ISP Router LAN
Physically connect one of OpenWrt’s WAN ports to a LAN port on the ISP gateway.
Configure OpenWrt WAN interface for DHCP to receive an IP from ISP router.

### Step 2: Configure LAN Bridge and Create VLANs
OpenWrt uses Linux bridges for LAN ports and wireless interfaces:
Under Network → Interfaces → LAN, create a bridge (br-lan) including all LAN ports and Wi-Fi.
Define VLANs for segmentation:
Go to Network → Switch (or Network → VLANs depending on device).
Create VLAN IDs (e.g., VLAN 10 for management, VLAN 20 guests, VLAN 30 IoT).
Assign physical ports and SSIDs to VLAN interfaces.

Example VLAN assignment:

VLAN ID	Interface	Ports / Wi-Fi
10	eth0.10	LAN1, LAN2 (management) & SSID1
20	eth0.20	LAN3 (guest) & SSID2
30	eth0.30	LAN4 (IoT devices) & SSID3

### Step 3: Create VLAN Interfaces
Under Network → Interfaces, add new interfaces:
Name them (e.g., LAN_man, LAN_guest, LAN_iot).
Assign VLAN subinterfaces (e.g., eth0.10, eth0.20).
Set static or DHCP IP addresses per VLAN subnet.

### Step 4: Configure Firewall Zones and Rules
Under Network → Firewall, create zones corresponding to VLAN interfaces:
For example, lan_man, lan_guest, and lan_iot.
Set zone forwarding and input/output policies:
Allow lan_man full internet and inter-zone access.
Restrict lan_guest and lan_iot from accessing lan_man.
Typically, guests and IoT have internet access only.

### Step 5: Disable DHCP Server on ISP Router if Possible (Optional)
For simpler subnet management, disable DHCP on ISP router and use OpenWrt as primary DHCP server.
Otherwise, accept double NAT and configure port forwards as needed.

### Step 6: Test Connectivity and Segmentation
Connect devices to respective VLAN SSIDs or LAN ports.
Test that guests and IoT cannot reach management devices.
Verify Internet access from all VLANs.

### Useful Links:
OpenWrt VLAN Setup Guide: https://openwrt.org/docs/guide-user/network/vlan/switch

OpenWrt Firewall Zones: https://openwrt.org/docs/guide-user/firewall/firewall_configuration

OpenWrt Bridge Interfaces Explanation: https://openwrt.org/docs/guide-user/network/network.bridge
