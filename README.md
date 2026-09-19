# Enterprise Multi-Department VLAN Network

Cisco Packet Tracer lab implementing a segmented enterprise network for
5 departments, with centralized DHCP relay, DNS, and FTP services, and
router-on-a-stick inter-VLAN routing.

## Objective
Design and configure a segmented enterprise LAN where each department
sits on its own VLAN, all inter-VLAN traffic is routed through a single
router, and core services (DHCP, DNS, FTP) are centralized on dedicated
servers instead of the router itself.

## Topology
- **Router0 (2911)**  router-on-a-stick, one sub-interface per VLAN
- **SW-CORE (3560)**  distribution switch, trunks to every access switch
- **4x Access switches (2960)**  SW-ACC1, SW-ACC2, SW-ACC3, SRV-SW
- **15x PCs**  3 per department across 5 departments
- **3x Servers**  dedicated DHCP, DNS, and FTP servers on VLAN 100

  <img src="file:///C:/Users/omari/OneDrive/Images/Screenshots/Capture%20d'%C3%A9cran%202026-09-19%20162907.png" alt="Texte alternatif" width="500">

## Departments & VLANs

| VLAN ID | Department  | Devices          | Subnet             |
|---------|-------------|------------------|---------------------|
| 10      | Sales       | PC0, PC1, PC2    | 192.168.10.0/24     |
| 20      | IT          | PC3, PC4, PC5    | 192.168.20.0/24     |
| 30      | HR          | PC6, PC7, PC8    | 192.168.30.0/24     |
| 40      | Finance     | PC9, PC10, PC11  | 192.168.40.0/24     |
| 50      | Engineering | PC12, PC13, PC14 | 192.168.50.0/24     |
| 99      | Native  | —       | 192.168.99.0/24     |
| 100     | Servers     | DHCP, DNS, FTP   | 192.168.100.0/24    |

## Servers (VLAN 100)

| Server | IP Address | Role |
|---|---|---|
| Server-DHCP | 192.168.100.10 | DHCP relay target for all 5 VLANs |
| Server-DNS | 192.168.100.20 | Name resolution for internal hosts |
| Server-FTP | 192.168.100.30 | File transfer service |

## Configuration

### Router sub-interfaces (Router0)
```
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 192.168.100.10
!
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip helper-address 192.168.100.10
!
interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip helper-address 192.168.100.10
!
interface GigabitEthernet0/0.40
 encapsulation dot1Q 40
 ip address 192.168.40.1 255.255.255.0
 ip helper-address 192.168.100.10
!
interface GigabitEthernet0/0.50
 encapsulation dot1Q 50
 ip address 192.168.50.1 255.255.255.0
 ip helper-address 192.168.100.10
!
interface GigabitEthernet0/0.100
 encapsulation dot1Q 100
 ip address 192.168.100.1 255.255.255.0
```

`ip helper-address` relays DHCP broadcasts from each VLAN to the
dedicated DHCP server on VLAN 100, since the server can't otherwise see
broadcast traffic from subnets it isn't directly attached to.

### Access switch trunk & access ports (example: SW-ACC1)
```
vlan 10,20,99
!
interface range Fa0/1-3
 switchport mode access
 switchport access vlan 10
!
interface range Fa0/4-6
 switchport mode access
 switchport access vlan 20
!
interface Fa0/7
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,99
```

### Core switch trunk configuration (SW-CORE)
```
vlan 10,20,30,40,50,99,100
!
interface range Fa0/1-4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40,50,99,100
```

### DHCP server pools (Server-DHCP → Services → DHCP)

| Pool | Default Gateway | DNS Server | Start IP | Subnet Mask |
|---|---|---|---|---|
| SALES | 192.168.10.1 | 192.168.100.20 | 192.168.10.11 | 255.255.255.0 |
| IT | 192.168.20.1 | 192.168.100.20 | 192.168.20.11 | 255.255.255.0 |
| HR | 192.168.30.1 | 192.168.100.20 | 192.168.30.11 | 255.255.255.0 |
| FINANCE | 192.168.40.1 | 192.168.100.20 | 192.168.40.11 | 255.255.255.0 |
| ENGINEERING | 192.168.50.1 | 192.168.100.20 | 192.168.50.11 | 255.255.255.0 |

## Testing

| Test | Result |
|---|---|
| PC → PC, same VLAN | Success |
| PC → PC, different VLAN (routed) | Success |
| `ipconfig /renew` on any PC | Receives correct IP from Server-DHCP |
| PC → Server-DNS / Server-FTP (VLAN 100) | Success |
| `ftp 192.168.100.30` from PC | Login prompt received |
| `show interfaces trunk` on all switches | All ports trunking, native VLAN 99 |
| `show ip dhcp pool` (if router pools used) | N/A — DHCP handled by dedicated server |

## What I Learned
- Trunk ports need matching encapsulation, mode, and native VLAN on
  **both ends** of the link — a mismatch on just one side silently
  breaks or flags the link.
- Centralizing DHCP on a dedicated server instead of the router
  requires `ip helper-address` on every routed sub-interface, since
  the server has no visibility into broadcast traffic from VLANs it
  isn't directly connected to.
- VLAN databases must be created (`vlan 10,20,...`) on every switch
  in the path — trunk "allowed vlan" lists don't implicitly create
  the VLANs themselves.

## Possible Extensions
- Replace router-on-a-stick with SVIs on a Layer-3 core switch
- Add port security (max MAC per port) on access switches
- Add inter-department ACLs (e.g. restrict HR from reaching Finance)
- Add a syslog server and forward switch/router logs to it
