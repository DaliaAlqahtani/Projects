**📘 Project Overview**

This project demonstrates a multi-site enterprise network topology that includes inter-VLAN routing via Router-on-a-Stick (ROAS) and dynamic routing using OSPF (Open Shortest Path First) as well as link aggregation. The topology simulates two separate branches (Branch-A and Branch-B), each implementing structured Layer 2 and Layer 3 designs for scalability and segmentation.

**🛠️ Technologies Used**

Cisco Routers and Switches (Simulated in GNS3)

OSPF Routing Protocol (Area 0)

Router-on-a-Stick (ROAS) Configuration

VLAN Segmentation (VLAN 10: SALES, VLAN 20: HR)

EtherChannel for switch-to-switch redundancy

802.1Q Trunking

**🧩 Topology Details**

*Branch-A*

R1 performs inter-VLAN routing for VLAN 10 and VLAN 20 via subinterfaces (f0/0.10, f0/0.20).

DSW1 (Distribution Switch) connects to R1 and connects to ACSW1 (Access Switch) using a trunked EtherChannel (Port-Channel) link.

VLANs:

VLAN 10: 10.10.10.0/24 (SALES)

VLAN 20: 10.10.20.0/24 (HR)

*Branch-B*

R2 mirrors the same ROAS setup as R1.

DSW2 connects to ACSW2 via EtherChannel trunk.

VLANs:

VLAN 10: 10.20.10.0/24 (SALES)

VLAN 20: 10.20.20.0/24 (HR)

*Inter-Site Communication*

R1 and R2 are connected over a point-to-point WAN link (10.0.0.0/30) and participate in OSPF Area 0 to dynamically advertise branch routes.

This enables full connectivity between departments (SALES ↔ SALES, HR ↔ HR) across branches.
