# Lab 6 — Optional End-to-End Testing

In this lab, you will be performing end-to-end testing of the multi-domain integration. You will be testing the reachability between the sites and ensuring following:

!!! warning "Prerequisite"
    Complete Labs 1 through 5 before starting this lab.

## Lab Objectives

After completing this lab, you will be able to:

- End-to-end propagation of SGTs from ISE to the network devices
- End-to-end reachability between the sites
- Policy enforcement between the endpoints
- The intention is to ensure deivces like laptops cannot access IoT devices, however they can access the servers which in turn can access the IoT devices. 

!!! note "CML Limitations"
    Current version of CML and Catalyst 9000 virtual switch could porvide inconsistent results in the lab environment. Full end-to-end testing works, but there might be some caveats. The steps described below require logging into the device directly using CML interface and opening a console session to the device. Due to caching issues, the commands you apply may be time sensitive as the ARP times out and end-to-end ping would fail. You can re-do the steps again to reproduce the results.

## Step 1: Connect to CML Console

From your RDP session, open a web browser and navigate to https://198.18.128.27 (or click on the CML bookmark on the browser) to access the Cisco Modeling Lab. Log in using the following credentials:

| Device Name | URL | Username | Password |
|---|---|---|---|
| Cisco Modeling Lab | [https://198.18.128.27](https://198.18.128.27) | `admin` | `C1sco12345` |


## Step 2: EDGE01 Configuration

EDGE01 will require sub steps, pleae follow them in the order below:

**Step 2.1 - CTS on EDGE01**
Configure CTS creds in EXEC mode, since it is EXEC, we can push through SAC

```cli
cts credentials id EDGE01.cisco.eu password C1sco12345
```

**Step 2.2 - DHCP on EDGE01**

Configure EDGE switch to be a local DHCP server

```cli
conf t
ip dhcp excluded-address vrf Campus 192.168.100.1 192.168.100.9
ip dhcp pool EMPLOYEES
 vrf Campus
 network 192.168.100.0 255.255.255.0
 default-router 192.168.100.1
 lease 0 8
end
```

**Step 2.3 - Mitigating ARP on Laptop Endpoint**
This is the manual workaround to mitigate ARP issue with CML C9KV

```bash
sudo ip neigh del 192.168.100.1 dev ens2 2>/dev/null
sudo ip addr flush dev ens2
sudo ip neigh flush dev ens2
sudo dhcpcd -k ens2 2>/dev/null
sudo dhcpcd -d -4 ens2
sudo ip neigh replace 192.168.100.1 lladdr 00:00:0c:9f:f6:f2 dev ens2 nud permanent
```

Confirm IP address was received from DHCP

```bash
ip addr show ens2
ip route
ip neigh show 
```

**Step 2.4 - Verify IP address on EDGE01**

Verify IP address was received from DHCP

```cli
show device-tracking database interface Gi1/0/1
show lisp instance-id 4100 ipv4 database | inc 192.168.100.10 # Use IP received from DHCP
show ip cef vrf Campus 192.168.100.10 # Use IP received from DHCP
show adjacency Vlan1021 detail | begin 192.168.100.10 # Use IP received from DHCP
```

**Step 2.5 - Configure SGT on EDGE01**

```cli
conf t
cts role-based sgt-map 192.168.100.10 sgt 40 
end
```

Verify SGT was configured

```cli
show cts role-based permissions
show cts role-based sgt-map all
```

## Step 3: EDGE02 Configuration

EDGE02 will require sub steps, pleae follow them in the order below:

**Step 3.1 - CTS on EDGE02**
Configure CTS creds in EXEC mode, since it is EXEC, we can push through SAC

```cli
cts credentials id EDGE02.cisco.eu password C1sco12345
```

**Step 3.2 - DHCP on EDGE02**

Configure EDGE switch to be a local DHCP server

```cli
conf t
ip dhcp excluded-address vrf Campus 192.168.110.1 192.168.110.9
ip dhcp pool SERVERS
 vrf Campus
 network 192.168.110.0 255.255.255.0
 default-router 192.168.110.1
 lease 0 8
end
```

**Step 3.3 - Mitigating ARP on Server Endpoint**
This is the manual workaround to mitigate ARP issue with CML C9KV

```bash
sudo ip neigh del 192.168.110.1 dev ens2 2>/dev/null
sudo ip addr flush dev ens2
sudo ip neigh flush dev ens2
sudo dhcpcd -k ens2 2>/dev/null
sudo dhcpcd -d -4 ens2
sudo ip neigh replace 192.168.110.1 lladdr 00:00:0c:9f:f5:82 dev ens2 nud permanent
```

Confirm IP address was received from DHCP

```bash
ip addr show ens2
ip route
ip neigh show
```

**Step 3.4 - Verify IP address on EDGE02**

Verify IP address was received from DHCP

```cli
show device-tracking database interface Gi1/0/1
show lisp instance-id 4100 ipv4 database | inc 192.168.110.10 # Use IP received from DHCP
show ip cef vrf Campus 192.168.110.10 # Use IP received from DHCP
show adjacency Vlan1022 detail | begin 192.168.100.10 # Use IP received from DHCP
```

**Step 3.5 - Configure SGT on EDGE02**

```cli
conf t
cts role-based sgt-map 192.168.110.10 sgt 20
end
```

Verify SGT was configured

```cli
show cts role-based permissions
show cts role-based sgt-map all
```

## Step 4: FIAB Configuration

Configure CTS creds in EXEC mode, since it is EXEC, we can push through SAC


**Step 4.1 - CTS on FIAB**

```cli
cts credentials id FIAB.cisco.eu password C1sco12345
```

**Step 4.2 - DHCP Server on FIAB**

```cli
conf t
ip dhcp excluded-address vrf IoT 192.168.200.1 192.168.200.9
ip dhcp pool CAMERAS
 vrf IoT
 network 192.168.200.0 255.255.255.0
 default-router 192.168.200.1
 lease 0 8
end
```

**Step 4.3 - Mitigating ARP on Camera Endpoint**
This is the manual workaround to mitigate ARP issue with CML C9KV

```bash
sudo ip neigh del 192.168.200.1 dev ens2 2>/dev/null
sudo ip addr flush dev ens2
sudo ip neigh flush dev ens2
sudo dhcpcd -k ens2 2>/dev/null
sudo dhcpcd -d -4 ens2
sudo ip neigh replace 192.168.200.1 lladdr 00:00:0c:9f:f5:33 dev ens2 nud permanent
```

Confirm IP address was received from DHCP

```bash
ip addr show ens2
ip route
ip neigh show
```

**Step 4.4 - Verify IP address on FIAB**

Verify IP address was received from DHCP

```cli
show device-tracking database interface Gi1/0/1
show lisp instance-id 4099 ipv4 database | inc 192.168.200.10 # Use IP received from DHCP
show ip cef vrf IoT 192.168.200.10 # Use IP received from DHCP
show adjacency Vlan1021 detail | begin 192.168.200.10 # Use IP received from DHCP
```

**Step 4.5 - Configure SGT on FIAB**

```cli
conf t
cts role-based sgt-map 192.168.200.10 sgt 30
end
```

Verify SGT was configured

```cli
show cts role-based permissions
show cts role-based sgt-map all
```


Once these steps are completed, you will be able to ping the endpoints and and validate following use case defined per the policy:

- Ping from Laptop to Server will be successful
- Ping from Laptop to Camera will be failed
- Ping from Server to Camera will be successful

**Continue to the [Conclusion](conclusion.md)**