[TOC]

# OAM

## **OAM Overview**

**Conception:** Operation administration and maintenance

**Protocol:** IEEE 802.3ah

**Purpose:** 

- **Operation:** The operation mainly completes the analysis, prediction, planning and configuration of daily network and business.
- **Maintenance:** Maintenance is mainly the daily operation activities of network and service testing and fault management. 

## **Terms**

- **Maintenance Entity (ME):** defines a relationship between two points of a transport path to which maintenance and monitoring operations apply.
- **Maintenance Entity Group(MEG):** The collection of one or more MEs that belong to the same transport path and that are maintained and monitored as a group are known as a Maintenance Entity Group.
- **Maintenance Point (MP):** A Maintenance Point (MP) is a functional entity that is defined at a node in the network and can initiate and/or react to OAM messages.
- **MEG End Point (MEP):** A MEG End Point (MEP) is one of the endpoints of an ME, and can initiate OAM messages and respond to them.
- **MEG Intermediate Point (MIP):** Between MEPs, there are zero or more intermediate points, called MEG Intermediate Points. A MEG Intermediate Point (MIP) is an intermediate point that does not generally initiate OAM frames (one exception to this is the use of AIS notifications) but is able to respond to OAM frames that are destined to it.

## **Application**

- **Fault Management **
- **Capability management**

## **Procedure**

- **Establish  Ethernet OAM connection**

The connected OAM entities report their OAM configuration and OAM capability information to each other by **information OAMPDU**.

The OAM entity decides to whether to establish a connection when received the configurations from the peer. And each peer has passed all configurations, Ethernet OAM will work.

After the Ethernet OAM connection is established, the OAM entities at both ends will periodically send Information OAMPDUs at a certain time interval to check whether the connection is normal. This interval is called the handshake message sending interval. If the OAM entity at one end does not receive the Information OAMPDU from the OAM entity at the opposite end within the connection timeout period, it is considered that the OAM connection is interrupted.

- **Link Monitoring**

Through event notification OAMPDU to monitor link.

- **Remote fault detection**

Through information OAMPDU to detect the remote fault.

- **Remote Loopback**

The OAM entity send the all other data packets (excluding OAMPDU data packets) to the peer at active mode. And the peer will send back that data packets in the same way.

# CFM

## **CFM Overview**

**Conception**: Connectivity Fault Management

**Protocol:**  IEEE 802.1ag

**Purpose:** The CFM protocol is used to detect, verify, isolate and report end-to-end Ethernet connection failures.

**CFM frames can be distinguished with normal frames through Ethernet type(0x8902) and destination MAC address**

## **Background**

Ethernet CFM is the end-to-end service instance.

- Active connection monitoring
- Fault verification
- Fault isolation

## **Terms**

- Maintenance Domain (MD)

MD indicates  the network that connection fault detection covered.

- Maintenance Domain Level (MD level)

There are  8 levels in total (0-7). The bigger number represents the higher level.

Different maintenance domains can be adjacent or nested, but cannot be crossed, and only higher-level maintenance domains can be used to nest lower-level maintenance domains.

- Maintenance array (MA)

Each maintenance set is a collection of some maintenance points in the maintenance domain.

A maintenance set serves a VLAN. The messages sent by the maintenance point in the maintenance center are tagged with the VLAN. At the same time, the maintenance point in the maintenance center can receive the messages sent by other maintenance points in the maintenance set.

- Maintenance point (MP)
- Maintenance endpoint(MEP)

Maintenance endpoints are directional and are divided into two types: Outward Maintenance Endpoint (Down MEP) and Inward Maintenance Endpoint (UP MEP). The direction of the maintenance endpoint indicates the position of the maintenance domain relative to the port. Among them, the outbound maintenance endpoint sends out packets through its port, and the inbound maintenance endpoint does not send out packets through its port, but sends out packets through other ports on the device.

- Maintenance intermediate point (MIP)

The maintenance intermediate point is located in the maintenance domain and cannot actively send out CFD PDUs, but can process and respond to CFD PDUs

***NOTE***

Similar to the maintenance endpoint, when the maintenance intermediate point receives a message higher than its own level, it will continue to forward it along the original path; when the maintenance intermediate point receives a message less than or equal to its own level, it will not forward it again , To ensure that packets in the low-level maintenance domain will not spread to the high-level maintenance domain.

MEP -> remote MEP

## **Protocol**

- continuity detection protocol
- loopback protocol
- link trace protocol

### **Continuity detection protocol**

- Use for fault detection, notification and regain
- The allowable "heartbeat" messages of each maintenance set are transmitted by MEP at a configurable fixed interval (3.3 ms, 10 ms, 100 ms, 1 second, 1 minute, 10 minutes)-one-way (no response required)
- Operator status of ports configured with MEP
- Cataloged by MIP at the same MD level and terminated by a remote MEP in the same MA

### **Loopback protocol**

- For fault verification-Ethernet Ping
- MEP can transmit unicast LBM to MEP or MIP in the same MA
- MEP can also transmit multicast LBM (defined by ITU-T Y.1731), in this case only MEP in the same MA will respond
- The receiver MP responds by converting the LBM to a unicast LBR and sending it back to the initiator MEP

### **Link trace protocol**

- For path discovery and fault isolation-Ethernet Traceroute
- MEP can transmit multicast messages (LTM) to discover MPs and paths to MIPs or MEPs in the same ME
- Each MIP and terminating MP along the path returns a unicast LTR to the initiator MEP

## **Implementation steps**

1. Run a connection check to proactively detect software or hardware failures.

2. When a failure is detected, it is verified using loopback, CCM database, and error database.

3. After verification, run traceroute to isolate it. You can also use multi-segment LBM to isolate faults.

4. If the isolated fault points to a virtual circuit, the OAM tool of this technology can be used to further isolate the fault; taking MPLS PW as an example, VCCV and MPLS ping can be used.



# Difference between OAM(802.3ah) and CFM(802.1ag)

## **Physical Media**

- IEEE 802.3ah is only used for 802.3 type networks

- IEEE 802.1ag can transimit 802 type frames in all physical media networks.

## **Loopback Detection**

- IEEE 802.3ah loops at the remote end, and the data is really looped back, affecting normal data transmission;
- IEEE 802.1ag doesn't loop, it just looks like request and response of ping, and doesn't affect the normal data transmission

## **Connectivity Detection**

- IEEE 802.3ah 's period is 1s and only checks whether the link is connected  but can't check  whether  the connections is expected.
- The IEEE 802.1ag cycle can be selected from a considerable range, both to check whether it is connected or not to detect unexpected connection errors.

## **Monitoring Range** 

- IEEE 802.3ah can only monitor a single link, not a shared link; 
- IEEE 802.1ag can be a single link, part or the entire network, and can monitor shared links