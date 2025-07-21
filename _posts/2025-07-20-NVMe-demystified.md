---
title: "NVMe demystified"
---
Fortunally, I was able to get an internship at Mangoboost, a company specializing in DPUs. My first assignment was to study **NVMe-oF** and **NVMe/TCP**, and to give a seminar on the topic.
This document summarizes the foundational knowledge I've studied, tailored for that seminar.
# Part 0: What is a network?

All network applications are fundamentally based on client-server model. The basic operation over client-server model is a *transaction.*

Basic flow of a transaction as:
1. If a client want a service, it initiates a transaction by sending a request to the server.
2. The server receives the request and processes it according to its own logic.
3. The server then sends back a response to the client.
4. The client takes the response and handles the result.
## Host vs Client-Server
Here we have to distinguish between **a Host and Client/Server roles.** A host is any devices connected to a network that can run processes. And a host can act as both a client and a server, even at the same time, through different processes or ports.
## How Hosts Connect to the Network
From the host's perspective, the network is just another I/O devices like a disk or keyboard. A host connects to the network through a network adapter (e.g., Ethernet card, Wi-Fi chip). This adapter is connected the system via the I/O bus. And the network adapter is physically connected to network infrastructure.

```plaintext
[ CPU & Memory ]
       |
    [ I/O Bus ]
       |
[ Network Adapter ] ───────▶ [ The Network ]
```

# Part 1: Storage Services
Storage refers to Non-Volatile memory devices, such as HDDs, SSDs and tape drives. We can categorizes storage systems into the following types:
## 1. DAS : Direct Attached Storage
This type of storage is directly attached to a single server, without going through a network.
```
[ Server ]
    |
    |  (Direct Connection: SATA / SAS / NVMe)
    |
[ Storage Device ] (HDD / SSD)
```

- Offers high performance due to direct access
- Not scalable: only one server can use the storage
- The server itself is responsible for managing the storage

## 2. NAS : Network Attached Storage
NAS is also known as **File Storage**, where files are accessed over a standard network(e.g. Ethernet).

```
[ Server ]       [ Server ]
    |                |
    +------ LAN -----+
            |
   [ NAS (File Server) ]
            |
        [ Storage ]
```

- Connects to servers via LAN
- A file server sits in between and manages the file system
- Scalable and easy to manage, especially when multiple servers need to share storage
- Generally slower than DAS or SAN due to file-level abstraction and network overhead

## 3. SAN : Storage Area Network
This type of storage is also called as **Block Storage**. SAN provides block-level storage over a dedicated high-speed network, such as Fibre Channel or iSCSI.

```
[ Server ]                          [ Server ]
    |                                   |
    +------ Fibre Channel / iSCSI ------+
                    |
            [ SAN Switch ]
	                |
         [ Block Storage Array ]
```


- Data is divided into fixed-size blocks, each with a unique access.
- Works like a locally attached disk, but over the network
- Offers high speed, reliability, and is highly scalable
- Requires specialized hardware (e.g., SAN switches, controllers)
- Typically used in enterprise environments or data centers

Data is splited into a certain size of a block, as seen in the parking lot. And each block has a unique address.
SAN is designed for a fast speed and stability, and it has a dependency with server.
Typically block storage has a I/O device, which is known as a **controller**.
## cf. Object Storage (Reference)

Unlike file or block storage, **object storage** manages data as **individual objects**, each containing:
- The data itself
- Metadata
- A unique identifier

# Part 2: Why NVMe has emerged?

![](https://i.imgur.com/JhMjiam.jpeg)

The photo above shows my Solid State Drives by SK hynix, installed in the PCIe slot on the motherboard(ASRock B550M PRO RS) of a desktop PC.

![](https://i.imgur.com/pxESAsS.jpeg)

The slot is labeled "PCIE2", and the SSD product is the Platinum P41. The interface specification listed on the label is "PCIe NVMe M.2."
**So what does this mean?**

In essence, **an SSD operates by running a COMMAND SET(=interface) over a TRANSPORT.**
For example:
- AHCI over SATA
- SCSI over SAS
- NVMe over PCIe

AHCI and SCSI were originally designed for spinning HDDs, not for the fast and parallel nature of SSDs. As a result, they became a performance bottleneck in modern storage systems.
For instance, AHCI only supports one queue with 32 commands, which is far too limited for today's high-throughput SSDs.
SCSI has similar constraints and overhead, making it inefficient for handling massive parallel I/O operations.
**This is where NVMe comes in.** NVMe is a protocol built specifically for SSDs, designed to fully exploit their speed, parallelism, and low latency. At first, NVMe was used over PCIe for direct-attached SSDS. But later it was extended to NVMe over Fabrics to enable remote access via Fibre Channel, RDMA or TCP/IP, bringing the gap between local and networked storage with minimal performance loss.

Before we move on, there's one more label to understand: M.2.
We've seen that the SSD is marked such as "PCIe NVMe M.2.". So, What exactly is M.2.?

To say briefly, M.2 specifies the **form factor** of the SSD.
![](https://cdn.mos.cms.futurecdn.net/QicCc9fkPdVEe38yx2QLhV-1200-80.jpg)
[Source](https://www.tomshardware.com/reviews/glossary-m2-definition,5887.html)

Being seen in the above picture, different SSDs have a different form, so the specification of the form is needed. This is the concept of the **form factor**.

2.5 inch SSDs use the SATA bus, and add-in cards use the PCIe bus. But M.2. SSDs can go either way, depending on the product. And some of the best SSDs use the NVMe interface!
To summarize, M.2. is just a form factor of SSDs, and M.2. SSDs can be SATA-based, PCIe-based with NVMe support or PCIe-based w/o NVMe support.

As we can see, my desktop is equipped with an M.2. SSD that uses the PCIe interface and supports the NVMe protocol.
