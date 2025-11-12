# Secure Home Network and Automation System Design

## Project Overview
Design and implement a secure, segmented home network architecture combined with a home automation platform that includes strong security and accessibility. Leverage OpenWrt on a consumer-grade router for advanced networking features and Proxmox VE for virtualization hosting Home Assistant.

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

## Diagram

<img width="839" height="679" alt="homeNetwork_20251112 drawio" src="https://github.com/user-attachments/assets/3c5de9c0-b54c-4920-bc1f-598ef0ccb1af" />

---

## Considerations

### Points to check when selecting a router:
- OpenWrt support: Official support for your model and revision (see [OpenWrt Table of Hardware](https://toh.openwrt.org/?view=normal))​
- Flash and RAM: Minimum 16MB flash and 128MB RAM; more is better for packages, logs, and remote access.​
- WPA3 capability: Secure modern Wi-Fi authentication.​
- Gigabit ports: Needed for high-speed LAN and WAN traffic.
- CPU Strength: Ensures router can handle multiple VLANs, firewall rules, and VPNs with low latency.
- Package space: Enough flash space for custom OpenWrt packages (e.g. wireguard, openvpn, adblock, luci-apps).
- Vendor openness: Avoid Broadcom chipsets which have poor open-source support.

  ### Project Router
  The Cudy WR3000S was selected as the core router platform for its combination of affordability, OpenWrt compatibility, and hardware features that fit the project’s security and automation goals. Some of the primary reasons for choosing this model:
  - OpenWrt Compatibility: The WR3000S is fully supported by OpenWrt v24+, making it easy to leverage advanced VLAN, firewall, and remote management features.​
  - WPA3 Support: The router hardware and OpenWrt build offer robust WPA3 wireless security out-of-the-box, which was a key requirement for protecting client network traffic.​
  - Ample Storage (128MB Flash): Unlike similar models with just 16MB of storage, WR3000S features 128MB NAND flash, ensuring space to install extra packages for VPN, DNS filtering, remote support, WireGuard/OpenVPN client/server, and other management tools needed for a future-proof install.​
  - Reliable Performance: With a 1.3GHz quad-core CPU, gigabit ports, and Wi-Fi 6, this router sustains high device counts and throughput, including for LAN and VLAN traffic requirements.​
  - Cost-effective and Compact: Delivers solid features at a lower cost than many enterprise solutions, and small size is ideal for a home setting.

### Proxmox Server selection tips:
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
 
  ### Project Proxmox VE Server
  The Beelink N100 Mini PC (also branded as Mini S12 Pro) was chosen primarily for its affordability and sufficient CPU and RAM capabilities to run the Home Assistant VM without extra overhead. Its Intel Alder Lake N100 quad-core processor at up to 3.4GHz and support for up to 16GB RAM provide a balanced platform tailored to the project’s modest resource needs.
  
  One limitation discovered during deployment was that this model only comes with a single built-in Gigabit Ethernet NIC. This presented a challenge because the project’s network design required multiple physical NICs to separate VLAN traffic, due to a limitation on the Cudy WR3000S ports.
  
  The single NIC constraint was overcome by using a USB to Ethernet adapter plugged into the Beelink N100. This inexpensive USB Ethernet solution effectively added a second NIC, allowing proper VLAN bridging and network segmentation despite the hardware limitations of the mini PC.

  This experience highlights the importance of NIC count for network routing and virtualization projects, and how pragmatic workarounds can adapt budget hardware to professional-grade network topologies.


### Network segmentation
The network was divided into three distinct segments to balance security, accessibility, and management efficiency:

- Management VLAN: This primary segment hosts critical devices such as personal computers, servers, and administration tools. Isolating management traffic ensures only trusted devices have full network access, reducing risk from less secure endpoints.
- Guest VLAN: A separate network for visitors provides internet access without exposing sensitive devices or data in the primary network, maintaining client privacy and preventing unauthorized access.
- IoT VLAN: IoT devices often have weaker security and pose a higher risk of compromise. Placing them on their own isolated VLAN mitigates the threat of lateral movement and prevents potential attacks from reaching the primary network and management devices.

Segmenting in this way enhances overall security by minimizing attack surfaces, improves network stability by limiting broadcast traffic within VLANs, and simplifies monitoring and applying tailored firewall rules for each segment.

---

# Stage 1: Installation

## Part 1: OpenWRT
Installing OpenWrt unlocks powerful networking features that consumer-grade firmware typically lacks. Below is a general guide to flashing OpenWrt on routers similar to the Cudy WR3000S.

### Step 1: Verify Router Compatibility
- Confirm your router model and hardware version is supported by OpenWrt.
- Check the OpenWrt Table of Hardware and search for your device to find compatible firmware images and notes.

### Step 2: Download the Correct OpenWrt Firmware
- Visit the OpenWrt Firmware Selector or the OpenWrt download page for your router.
- Download the correct release image for your router model and hardware revision.
- For Cudy WR3000S, ensure you select images built for that specific device.

### Step 3: Prepare for Installation
- Connect your computer to the router via Ethernet to avoid interruptions during the process.
- Set a static IP on your PC aligned with the router’s default management subnet if necessary (e.g., 192.168.1.x).

### Step 4: Flash the Firmware
- Access the router’s current administration interface (usually at http://192.168.1.1).
- Locate the firmware upgrade section.
- Upload the OpenWrt firmware image.
- Confirm and wait patiently for the router to flash and reboot. Interrupting this process can brick the device.

### Step 5: Initial Access and Configuration
- After reboot, connect to the new OpenWrt interface at http://192.168.1.1.
- No password is set initially; set a strong root password immediately.
- Begin configuring network settings including VLANs, Wi-Fi security (WPA3), firewall zones, and packages for remote management.

### Useful Resources
- OpenWrt Official Installation Guide: https://openwrt.org/docs/guide-quick-start/flashing
- Cudy WR3000S OpenWrt Community Discussions: https://forum.openwrt.org/t/openwrt-on-cudy-wr3000s/233708
- OpenWrt Firmware Selector: https://firmware-selector.openwrt.org/
- OpenWrt Table of Hardware (Compatibility): https://toh.openwrt.org/?view=normal

---

## Part 2: Proxmox
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

### Step 5: Enable Automatic Power-On
- While in BIOS/UEFI, find settings like "Restore on AC Power Loss" or "AC Back".
- Set it to "Power On" or "Always On" so that the device automatically powers on after power loss.

Note: This setting ensures your Proxmox server will automatically start without manual intervention after power outages or restarts.

Save and Exit BIOS/UEFI Settings

### Step 6: Install Proxmox VE
- Boot from the USB drive.
- Follow on-screen prompts:
  - Choose the target disk.
  - Set the country, timezone, and keyboard layout.
  - Define the admin password and network settings.
- Finish installation and remove the USB when prompted.
- Reboot the PC; it should now boot into Proxmox automatically.

### Step 7: Post-Installation Setup
- Log in via the web interface at https://[your-ip]:8006.
- Complete network configuration, storage setup, and VM deployment as needed.

### Resources:
- [Proxmox VE Official Installation Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#_installation)
- [Official Etcher Website](https://etcher.balena.io/)

---

# Stage 2: Network & Server Configurations

## Part 1: OpenWRT bridges, VLANs, & firewall

When placing OpenWrt behind an ISP-provided router (in a double NAT or routed mode) for simplicity and redundancy, careful network and firewall configuration is essential to maintain segmentation and security.

### Step 1: Connect OpenWrt WAN to ISP Router LAN
- Physically connect one of OpenWrt’s WAN ports to a LAN port on the ISP gateway.
- Configure OpenWrt WAN interface for DHCP to receive an IP from ISP router.

### Step 2: Configure LAN Bridge and Create VLANs
- OpenWrt uses Linux bridges for LAN ports and wireless interfaces:
  - Under Network → Interfaces → LAN, create a bridge (br-lan) including all LAN ports and Wi-Fi.
- Define VLANs for segmentation:
  - Go to Network → Switch (or Network → VLANs depending on device).
  - Create VLAN IDs (e.g., VLAN 10 for management, VLAN 20 guests, VLAN 30 IoT).
  - Assign physical ports and SSIDs to VLAN interfaces.

Example VLAN assignment:

| VLAN ID | Interface | Ports / Wi-Fi                   |
| ------- | --------- | ------------------------------- |
| 10      | eth0.10   | LAN1, LAN2 (management) & SSID1 |
| 20      | eth0.20   | LAN3 (guest) & SSID2            |
| 30      | eth0.30   | LAN4 (IoT devices) & SSID3      |

### Step 3: Set Up Wireless Networks for Each VLAN
- Navigate to Network → Wireless.
- For each radio interface (2.4GHz and 5GHz), create separate Wi-Fi networks for:
  - Management (e.g., SSID: HomeLAN) associated with VLAN 10
  - Guest (e.g., SSID: GuestWiFi) associated with VLAN 20
  - IoT (e.g., SSID: IoTNet) associated with VLAN 30
    * Note: IOT devices typically only operate on 2.4GHz networks.

- Configure each wireless network with:
  - Mode: Access Point (AP)
  - Network: Select the corresponding VLAN interface (lan_man, lan_guest, lan_iot)
  - Security: Use WPA3 or WPA2 with a strong passphrase
- Save, apply, and enable each wireless network configuration.

### Step 4: Create VLAN Interfaces
- Under Network → Interfaces, add new interfaces:
  - Name them (e.g., LAN_man, LAN_guest, LAN_iot).
  - Assign VLAN subinterfaces (e.g., eth0.10, eth0.20).
  - Set static or DHCP IP addresses per VLAN subnet.

### Step 5: Configure Firewall Zones and Rules
- Under Network → Firewall, create zones corresponding to VLAN interfaces:
  - For example, lan_man, lan_guest, and lan_iot.
- Set zone forwarding and input/output policies:
  - Allow lan_man full internet and inter-zone access.
  - Restrict lan_guest and lan_iot from accessing lan_man.
  - Typically, guests and IoT have internet access only.

### Step 6: Disable DHCP Server on ISP Router if Possible (Optional)
- For simpler subnet management, disable DHCP on ISP router and use OpenWrt as primary DHCP server.
- Otherwise, accept double NAT and configure port forwards as needed.

### Step 7: Test Connectivity and Segmentation
- Connect devices to respective VLAN SSIDs or LAN ports.
- Test that guests and IoT cannot reach management devices.
- Verify Internet access from all VLANs.

### Useful Links:
- OpenWrt VLAN Setup Guide: https://openwrt.org/docs/guide-user/network/vlan/switch
- OpenWrt Firewall Zones: https://openwrt.org/docs/guide-user/firewall/firewall_configuration
- OpenWrt Bridge Interfaces Explanation: https://openwrt.org/docs/guide-user/network/network.bridge

## Part 2: Home Assistant VM
This section explains how to configure the Home Assistant virtual machine (VM) in Proxmox to access both the management VLAN (VLAN 10) and the IoT VLAN (VLAN 30) using a USB-to-Ethernet adapter as a second network interface.

### Step 1: Prepare Proxmox Network Bridges for VLANs
1. Create Linux Bridges:
   - Log in to the Proxmox web UI.
   - Go to Datacenter → Node → Network.
   - Create or verify existing Linux bridges mapped to physical NICs assigned to VLANs:
     - Example: vmbr10 mapped to eth0.10 (management VLAN 10)
     - Example: vmbr30 mapped to eth1.30 (IoT VLAN 30 via USB-Ethernet adapter)

2. Configure VLAN Interfaces:
   - Use Proxmox shell or helper scripts (e.g., pvecm, networking commands) to add VLAN subinterfaces if necessary.

### Step 2: Download and Install Home Assistant VM
1. Download Home Assistant OS VM Image:
   - Obtain the latest QCOW2 or OVA image from the official Home Assistant website: https://www.home-assistant.io/installation/

2. Upload Image to Proxmox Storage:
   - Upload the VM disk image to a Proxmox storage location via the web UI or SCP.

3. Create a New VM:
   - In Proxmox, create a new VM with appropriate CPU, RAM (8GB+ recommended), and disk size.
   - During setup, select the uploaded Home Assistant disk image for the boot disk.
   - Allocate two network interfaces connected to vmbr10 and vmbr30 respectively.

4. Use VirtIO Drivers:
   - Choose virtio for network and disk to optimize performance.

### Step 3: Attach Network Interfaces to Home Assistant VM
- Confirm VM has two NICs assigned:
  - NIC 1 attached to vmbr10 (management VLAN).
  - NIC 2 attached to vmbr30 (IoT VLAN via USB Ethernet adapter).

### Step 4: Configure Networking Inside Home Assistant
- Configure Home Assistant network to recognize two interfaces, assign static IPs or use DHCP within each VLAN subnet.
- Verify routing and firewall rules permit required traffic.

### Step 5: Test Connectivity
- Confirm Home Assistant VM can access both VLANs.
- Ping devices on management and IoT networks.
- Verify home automation functions across segmented networks.

### Notes and References
- USB Ethernet adapter support depends on Proxmox host recognizing the device.
- Ensure VLAN tagging on USB adapter is properly configured in Proxmox bridge.
- See Proxmox network docs and Home Assistant VM installation guide for details:
  - https://pve.proxmox.com/wiki/Network_Configuration
  - https://www.home-assistant.io/installation/alternative#kvm-qemu
 
---

# Phase 3: Testing and Validation

1. Verify IoT Device Connectivity:
   - Connect IoT devices to the IoT VLAN (VLAN 30) Wi-Fi or Ethernet ports.
   - Confirm devices receive appropriate IP addresses from the OpenWrt DHCP server for VLAN 30.
   - Test internet connectivity from IoT devices.

2. Add IoT Devices to Home Assistant:
   - Using the Home Assistant UI, discover and add the IoT devices.
   - Ensure Home Assistant can communicate with devices over the IoT VLAN network.
   - Validate device status updates and control commands function correctly.

3. Control from Management VLAN:
   - From devices connected on the management VLAN (VLAN 10), access the Home Assistant dashboard.
   - Issue commands to IoT devices and verify responses.
   - Confirm segmentation rules prevent direct access to IoT devices outside of Home Assistant.

4. Network Isolation Verification:
   - Attempt to ping IoT devices directly from management VLAN endpoints — should be allowed by firewall rules.
   - Ensure IoT devices cannot initiate communication with management devices except through Home Assistant.
  
---

# Phase 4 (Optional): Secure Remote Management

To enable secure, encrypted remote access to the home network and Home Assistant interface without exposing ports to the public internet, Tailscale was installed both on a dedicated VM inside Proxmox and on the OpenWrt router.

Key Benefits:
- Zero-configuration VPN: Tailscale creates a mesh VPN overlay using the WireGuard protocol, enabling devices to securely connect across the internet as if on the same local network.
- Encrypted and authenticated: All traffic is end-to-end encrypted and authenticated using identity-based keys.
- Easy access: Allows administrator devices to reach Home Assistant and network management interfaces remotely without opening firewall ports or configuring complex VPN servers.

Installation Highlights:
- On Proxmox VM:
  - A lightweight Debian based Linux VM was created specifically to run Tailscale in client mode.
  - This VM securely connects to the Tailscale network, providing a gateway for remote administrative tasks and relay for services that require external accessibility.
- On OpenWrt Router:
  - The Tailscale package was installed via OpenWrt’s opkg system.
  - Running on the router itself allows the entire home network to be reachable through the Tailscale mesh.
  - Ensures secure tunnel for management VLAN and IoT VLAN without exposing the entire router to the open internet.

Usage:
- Administrators log into their Tailscale accounts on laptops or mobile devices.
- Via the Tailscale network, they can SSH into the Proxmox host, access the Home Assistant UI, or manage the OpenWrt router’s LuCI interface securely.

Additional Resources:
- Tailscale Official Site: https://tailscale.com
- OpenWrt Tailscale Installation Guide: https://openwrt.org/docs/guide-user/services/tailscale
- Running Tailscale on Proxmox VM: https://tailscale.com/kb/1135/install-linux/

---

# Conclusion and Lessons Learned
This project demonstrated the benefits and complexities of designing a secure, segmented home network with integrated home automation. Some of the key lessons learned include:
- IoT Device Challenges: Many IoT devices use diverse, often proprietary ports and protocols, complicating strict network segmentation and firewall configuration. Understanding each device’s network requirements upfront helps avoid connectivity issues and ensures smoother integration with Home Assistant.
- Hardware Limitations: Limited physical network interfaces on budget hardware, such as the single NIC on the Beelink N100, require creative solutions like USB-to-Ethernet adapters to fulfill networking needs without sacrificing segmentation or performance.
- Vendor Compatibility Issues: IoT device compatibility is variable, with some vendors employing security or protocol implementations that impede seamless operation in segmented environments. Thorough research and testing of devices before deployment can prevent later headaches.
- Balancing Security and Usability: Effective segmentation enhances security but requires carefully crafted firewall rules and routing to maintain usability, especially for automation and device discovery.

By proactively addressing these challenges, future deployments can achieve robust security without sacrificing accessibility or convenience, resulting in a resilient smart home ecosystem.
