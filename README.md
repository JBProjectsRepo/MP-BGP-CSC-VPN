###  MP-BGP CSC VPN and PYATS ###

This project aims to show how to configure a MP-BGP Carrier Suport Carrier VPN between clients in a network. Also, it will server as a start-point for another project that uses **Cisco python Library Pyats** in order to automate configuration of a new clients and to test the network against modifications on the configurations.

All the networking is done under EVE-NG Community Edition.

### GOAL ###

The idea is to have a network where **Client-1** and **Client-2** have two sites each, that communicate over a VPN which passes through two Service Providers, **Main SP** and **Secondary SP**. Also,** Client-1 network 192.168.11.0** should be able to communicate with **Client-2 network 21.21.21.0**. Finally, **Client-3** VPN passes only through **Main SP** and can communicate only with **Client-3** sites.

### TOPOLOGY ###

![](https://pandao.github.io/editor.md/images/logos/editormd-logo-180x180.png)

### EXPECTED ###

- **Client-1** in **R1** can receive only Client-1 routes
- **Client-1** in **R13** can receive Client-1 and 21.21.21.0/24 (Client-2) routes
- **Client-2** in **R17** can receive only Client-2 routes
- ** Client-2** in **R16** can receive Client-2 and 192.168.11.0/24 (Client-1) routes
- **Client-3** can receive only Client-3 routes
- **MAIN-SP** **R6**, **R7** and **R8** can receive only internal routes (ospf) from MAIN-SP
- **MAIN-SP** **R5** and **R9** can receive routes from Secondary-SP via VRF, from Client-3 via VRF and internal routes from MAIN-SP (ospf)
- **Secondary-SP** **R3**, **R4**, **R10** and **R11** can receive only internal routes (ospf) from Secondary-SP
- **Secondary-SP** **R2** and **R12** have routes from Secondary-SP and from Client-1, 2 via VRF

**Note:** This is important in order to guide what we expect in our automated tests with PYATS.

### REFERENCES ###

https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/mp_ias_and_csc/configuration/15-s/mp-ias-and-csc-15-s-book/mp-carrier-bgp.html

https://packetlife.net/blog/2013/sep/26/vrf-export-maps/

### STEPS ####

+ Config **OSPF** in MAIN SP AS 65000 routers **R5, R6, R7, R8, R9** internal interfaces (172.29.x.x) and Loopbacks
```
router ospf 1
				router-id a.a.a.a
				network n.n.n.n m.m.m.m
```

	+ Where:
		+ a.a.a.a can be any of the router internal interfaces
		+ n.n.n.n is the desired interface network and m.m.m.m its mask
		+ Note: For loopbacks, you should also add the configuration below in order to avoid it being distributed as a /32 network:
```
interface LoX
				ip ospf network point-to-point
```

- Config **MPLS** in MAIN SP AS 65000 routers **R5, R6, R7, R8, R9** internal interfaces (172.29.x.x)
```
mpls ldp router-id int a/a
int y
				mpls ip
```
	+ Where:
		+ int a/a can be any of the router internal interfaces
		+ int y are all of the router internal interfaces

- Create the** VRF** for SP-2 in MAIN-SP **R5** and **R9 **
```
#R5
ip vrf vpn_sp2
				rd 65021:100
				route-target export 65021:100
				route-target import 65022:100
interface e0/0
				ip vrf forwarding vpn_sp2
				ip address 189.100.4.5 255.255.255.0
#R9
ip vrf vpn_sp2
				rd 65022:100
				route-target export 65021:100
				route-target import 65022:100
interface eth0/2
				ip vrf forwarding vpn_sp2
				ip address 189.100.9.9
 ```

-  Config **BGP** Between MAIN-SP **R5** and **R9** and activate SP-2 VPN
```
#R5
router bgp 65000
				bgp router-id 5.5.5.5
				neighbor 9.9.9.9 remote-as 65000
				neighbor 9.9.9.9 update-source Lo0
				address-family ipv4
								neighbor 5.5.5.5 activate
								neighbor 5.5.5.5 send-community extended
								exit-address-family
 				address-family vpnv4
  								neighbor 9.9.9.9 activate
  								neighbor 9.9.9.9 send-community extended
 								exit-address-family
#R9
router bgp 65000
				bgp router-id 9.9.9.9
				neighbor 5.5.5.5 remote-as 65000
				neighbor 5.5.5.5 update-source Lo0
				address-family ipv4
								neighbor 5.5.5.5 activate
								neighbor 5.5.5.5 send-community extended
								exit-address-family
				address-family vpnv4
								neighbor 5.5.5.5 activate
								neighbor 5.5.5.5 send-community extended
								exit-address-family
```
- Config **BGP** Between MAIN-SP **R5** and** R9** and SP-2 **R4** and **R10**
```
#R5
router bgp 65000
				address-family ipv4 vrf vpn_sp2
 								neighbor 189.100.4.4 remote-as 65021
 								neighbor 189.100.4.4 activate
#R4
router bgp 65021
				neighbor 189.100.4.5 remote-as 65000
#R9
router bgp 65000
				address-family ipv4 vrf vpn_sp2
 								neighbor 189.100.9.10 remote-as 65022
 								neighbor 189.100.9.10 activate
#R10
router bgp 65021
				neighbor 189.100.9.9 remote-as 65000
```

- Config **OSPF** in SP-2 AS 65021 and 65022  **R2, R3, R4, R10, R11, R12** internal interfaces (172.21.x.x) and loopbacks.
```
router ospf 1
				router-id a.a.a.a
				network n.n.n.n m.m.m.m
```

	+ Where:
		+ a.a.a.a can be any of the router internal interfaces
		+ n.n.n.n is the desired interface network and m.m.m.m its mask
		+ Note: For loopbacks, you should also add the configuration below in order to avoid it being distributed as a /32 network:
```
interface LoX
				ip ospf network point-to-point
```

- Config redistribute between **BGP** and **OSPF** in **R10** and **R4**
```
#R4
router ospf 1
				redistribute bgp 65021 metric-type 1 subnets
router bgp 65021
				redistribute ospf 1 metric 1 match internal external 1 external 2
#R10
router ospf 1
				redistribute bgp 65022 metric-type 1 subnets
router bgp 65022
				redistribute ospf 1 metric 1 match internal external 1 external 2
```

- Activate **MPLS** in all nodes of Secondary-SP (**Except links with MAIN-SP** because we are using **MP-EBGP** to connect Secondary-SP to MAIN-SP, **and links with Clients**)
```
mpls ldp router-id int a/a
int y
				mpls ip
```
	+ Where:
		+ int a/a can be any of the router internal interfaces
		+ int y are all of the router internal interfaces
	

- Configure **BGP** neighborhood beetwen **R2** and **R12** and allow them to exchange **vpn routes**
```
#R12
router bgp 65022
				neighbor 2.2.2.2 remote-as 65021
				neighbor 2.2.2.2 update-source Loopback0
				address-family ipv4
								neighbor 2.2.2.2 activate
								neighbor 2.2.2.2 send-community extended
								exit-address-family
				address-family vpnv4
								neighbor 2.2.2.2 activate
								neighbor 2.2.2.2 send-community extended
								exit-address-family
#R2
router bgp 65022
				neighbor 12.12.12.12 remote-as 65022
				neighbor 12.12.12.12 update-source Loopback0
				address-family ipv4
								neighbor 12.12.12.12 activate
								neighbor 12.12.12.12 send-community extended
								exit-address-family
				address-family vpnv4
								neighbor 12.12.12.12 activate
								neighbor 12.12.12.12 send-community extended
								exit-address-family
```

- Configure **EBGP Multihop** in order to allow BGP neighborhood between **R2** and **R12**
```
#R2
router bgp 65021
				neighbor 12.12.12.12 ebgp-multihop 255
#R12
router bgp 65022
				neighbor 2.2.2.2 ebgp-multihop 255
```

- Create **VRF** for Client-1 and Client-2 in Secondary-SP **R2** and **R12** (Here we will not allow communication between Client-1 and 2 yet)
```
#R2
ip vrf vpn_c1
				rd 65011:100
				route-target export 65011:100
				route-target import 65012:100
ip vrf vpn_c2
				rd 65221:100
				route-target export 65221:100
				route-target import 65222:100
int eth0/0
				ip vrf forwarding vpn_c1
				ip address 100.1.0.2 255.255.255.0
int eth0/3
				ip vrf forwarding vpn_c2
				ip address100.2.0.2 255.255.255.0
#R12
ip vrf vpn_c1
				rd 65012:100
				route-target export 65012:100
				route-target import 65011:100
ip vrf vpn_c2
				rd 65222:100
				route-target export 65222:100
				route-target import 65221:100
int eth0/1
				ip vrf forwarding vpn_c1
				ip address 100.1.2.12 255.255.255.0
int eth0/2
				ip vrf forwarding vpn_c2
				ip address 100.2.1.12 255.255.255.0
 ```

- Configure **BGP** neighborhood beetwen **R2** and Clients-1 and 2 and** R12** and Clients-1 and 2
```
 #R2
router bgp 65021
				address-family ipv4 vrf vpn_c1
								neighbor 100.1.0.1 remote-as 65011
								neighbor 100.1.0.1 activate
								exit-address-family
				address-family ipv4 vrf vpn_c2
								neighbor 100.2.0.16 remote-as 65221
								neighbor 100.2.0.16 activate
								exit-address-family
  #R1
router bgp 65011
				neighbor 100.1.0.2 remote-as 65021
				network 10.0.0.0 mask 255.255.255.0
#R16
router bgp 65221
				neighbor 100.2.0.2 remote-as 65021
				network 21.21.21.21 mask 255.255.255.0
				network 23.23.23.23 mask 255.255.255.0
 #R12
router bgp 65022
				address-family ipv4 vrf vpn_c1
								neighbor 100.1.2.13 remote-as 65012
								neighbor 100.1.2.13 activate
								exit-address-family
				address-family ipv4 vrf vpn_c2
								neighbor 100.2.1.17 remote-as 65222
								neighbor 100.2.1.17 activate
								exit-address-family
 #R13
router bgp 65012
				neighbor 100.1.2.12 remote-as 65022
				network 192.168.0.0 mask 255.255.255.0
				network 192.168.11.0 mask 255.255.255.0
#R17
router bgp 65222
				neighbor 100.2.1.12 remote-as 65022
				network 22.22.22.22 mask 255.255.255.0
```

- Configure **R4, R5** and **R9, R10** to exchange bgp labels:
```
#R4
router bgp 65000
				address-family ipv4 vrf vpn_sp2
								neighbor 189.100.4.4 send-label
#R5
router bgp 65021
				neighbor 189.100.4.5 send-label
#R9
router bgp 65000
				address-family ipv4 vrf vpn_sp2
								neighbor 189.100.9.10 send-label
#R10
router bgp 65021
				neighbor 189.100.9.9 send-label
```

- Allow **R2** to **export** only **21.21.21.21** from  vpn_c2 in **R2** and to **import** **192.168.11.11** from **R12**:
```
#R2
ip prefix-list pfx_c2_to_c1 seq 5 permit 21.21.21.0/24
route-map Client2_to_Client1 permit 10
				match ip address prefix-list pfx_c2_to_c1
				set extcommunity rt 65221:200 additive
ip vrf vpn_c2
				export map Client2_to_Client1
				route-target import 65012:200
```

- Allow **R12** to **export** only **192.168.11.11** from vpn_c1 in **R12** and to **import** **21.21.21.21** from **R2**
```
#R12
ip prefix-list pfx_c1_to_c2 seq 5 permit 192.168.11.0/24
route-map Client1_to_Client2 permit 10
				match ip address prefix-list pfx_c1_to_c2
				set extcommunity rt 65012:200 additive
ip vrf vpn_c1
				export map Client1_to_Client2
				route-target import 65221:200
```

- Configure Client-3 **VRF** and **BGP address-family** in **R5** and **R9**
```
#R9
ip vrf vpn_c3
				rd 65032:100
				route-target export 65032:100
				route-target import 65031:100
interface Ethernet0/3
				ip vrf forwarding vpn_c3
				ip address 100.3.1.9 255.255.255.0
router bgp 65000
				address-family ipv4 vrf vpn_c3
								neighbor 100.3.1.14 remote-as 65032
								neighbor 100.3.1.14 activate
#R5
ip vrf vpn_c3
				rd 65031:100
				route-target export 65031:100
				route-target import 65032:100
interface Ethernet0/2
				ip vrf forwarding vpn_c3
				ip address 100.3.0.5 255.255.255.0
router bgp 65000
				address-family ipv4 vrf vpn_c3
								neighbor 100.3.0.15 remote-as 65031
								neighbor 100.3.0.15 activate
```
