**📘 Project Overview**

This project demonstrates Infrastructure as Code principles by automating Cisco switch configurations using Ansible. The automation streamlines the deployment of VLANs, DHCP security, and port security across multiple switches in a simulated enterprise environment.

**🧩 Topology Details**

- **Management VLAN (VLAN 10):** 10.10.10.0/24 - contains the Ansible control node as well as the managed nodes ACSW1 and ACSW4
- **HR VLAN (VLAN 20):** 10.20.20.0/24 - managed by ACSW1 and ACSW2
- **Marketing VLAN (VLAN 30):** 10.30.30.0/24 - managed by ACSW3 and ACSW4
- **Ubuntu Control Node:** running Ansible for automation
- **Distribution Switch:** Default gateway with inter-VLAN routing

**🛠️ Technologies Used**

**Ansible**-For configuration management and automation
**Cisco IOS**-As the network devices operating system
**SSH/Pylibssh**-For secure device connectivity
**YAML**-For playbook and inventory configurations
**AWS EC2**-The cloud infrastructure hosting as the GNS3 Server
**GNS3 Ubuntu Docker Container**-As the containerized control node

**🔧Automated Configurations**

**ACSW1 (One of HR Switchs)**
✅ VLAN 20 creation and naming
✅ DHCP snooping implementation
✅ Port security on access ports
✅ Trunk configuration with allowed VLANs

**ACSW4 (One of Marketing Switchs)**

✅ VLAN 30 creation and naming
✅ DHCP snooping implementation
✅ Port security on access ports
✅ Trunk configuration with allowed VLANs

**🔒 Security Implementations for this project**

- **DHCP Snooping:** to prevent rogue DHCP requests
- **Port Security:** to limit MAC addresses allowed per port (maximum 1)
- **Sticky MAC Learning:** to automatically secure learned MAC addresses
- **Violation Handling:** to restrict traffic on security violations

**✂️ Network Segmentation**

- **Access Ports:** e0/1-3, e1/0-3, e2/0-3, e3/0-3 configured for respective VLANs
- **Trunk Ports:** e0/0 configured with 802.1Q encapsulation
- **Native VLAN:** VLAN 100 for untagged traffic
- **Management Access:** VLAN 10 allowed on trunks for control node connectivity

**📋 Prerequisites**

- Ubuntu control node with Ansible installed
- SSH connectivity to target switches
- Cisco switches with management IPs configured
- Basic switch configuration (SSH, user accounts, enable passwords)

**🤔 How Ansible Works?**
Ansible mainly needs two types of components: **Inventory** and **Playbook.**
**Inventory Management:** Defines the target hosts (e.g., servers, switches, routers) and their connection details, such as IP addresses, SSH credentials, or connection types.
**Playbook Execution:** Executes a series of automation tasks (written in YAML) to configure, deploy, or manage systems on the defined targets.
