# Host PLDM Agent over PCIe MMBI

Author: linyuanhan

Created: November 11, 2026

Status: Design draft

## Problem Description

A BMC's out-of-band management view does not necessarily include the logical
device relationships, software configuration, or runtime state maintained by
the host operating system. This proposal provides a way to make that information
available through standard management protocols and integrate it with the BMC's
existing inventory, monitoring, and Redfish services.

The goal is to expose inventory data, sensor readings, Redfish resources, and
management event notifications while the host OS is running. The proposed Host
PLDM Agent runs MMBI, MCTP, and PLDM processing in userspace. An ASPEED PCIe shared
memory channel provides the hardware access and notifications used by MMBI;
MCTP carries PLDM messages between the host and BMC. Extensions to the BMC's
existing MCTP and PLDM services provide the corresponding management access.

Local development and validation are also part of the proposal. Unix domain
sockets (UDS) carry MCTP packets between the agent and a local protocol client,
while an HTTP interface provides direct access to Redfish resource content.
This allows contributors to exercise resource semantics, PLDM/RDE exchanges,
and Binary Encoded JSON (BEJ) conversion without a target BMC or PCIe MMBI link.

## Background and References

### Hardware and software references

The ASPEED Host-BMC PCIe function provides host access to BMC shared memory and
notifications in both directions. The reference drivers have distinct roles:

| Side | Reference | Relevant functionality |
| --- | --- | --- |
| BMC | [aspeed-bmc-dev.c][1] | BMC-side PCIe memory mapping and notification support |
| Host OS | [aspeed-host-bmc-dev.c][2] | Host-side PCIe device access and shared memory mapping |

The [BMC eSPI MMBI implementation][3] provides a reference for MMBI communication
and its kernel-to-userspace interface. The proposal adapts that communication
model to the existing PCIe hardware channel while preserving compatibility with
the BMC MMBI application ABI.

The BMC software starting points are the [mctpd][4] and [pldmd][5] reference
repositories under the `ocp-hm-openbmc-opf-ami` namespace, together with their
related MCTP libraries. This document refers to those repositories as OCP mctpd
and OCP pldmd. The proposed MCTP extensions provide MMBI transport support; the
pldmd extensions access host management capabilities, receive events, and
integrate the results with existing BMC services.

### Protocol layering

The protocol references are [MMBI][6], [MCTP over MMBI][7],
[PLDM Platform][8], [PLDM FRU][9], and
[PLDM for Redfish Device Enablement (RDE)][10]. Their roles are:

| Layer | Responsibility |
| --- | --- |
| PCIe hardware channel | Shared memory access and notifications |
| MMBI | A communication interface between host and BMC using shared buffers |
| MCTP over MMBI | The transport binding for MCTP packets carried over MMBI |
| MCTP | Management message transport, including fragmentation and reassembly where needed |
| PLDM | Capability discovery, inventory and monitoring access, RDE operations, and event exchanges |

The selected transport binding is MCTP over MMBI. PCIe supplies the underlying
hardware channel; this is distinct from the MCTP over PCIe VDM binding.

## Requirements

### Functional requirements

| Capability | Host responsibility | BMC responsibility |
| --- | --- | --- |
| PLDM capability discovery | Provide PLDM terminus identification and supported management capabilities | Identify the host terminus and determine which capabilities are available |
| FRU | Provide device inventory data available to the host | Retrieve the data and integrate it with BMC inventory services |
| Sensors | Provide sensor descriptions, readings, and states through PLDM Platform | Discover and access host sensors through BMC monitoring services |
| RDE | Provide Redfish resources accessible through PLDM | Act as the RDE requester and integrate resource access with the BMC Redfish service |
| PLDM events | Report management changes through the PLDM Platform event mechanism | Receive supported event classes and perform the corresponding management actions |

The initial functional scope is information retrieval and monitoring.
Configuration changes, device control, and firmware updates can be considered
in later extensions. Event classes and their handling depend on the capabilities
supported by both sides.

### Architectural requirements

1. **Host protocol processing in userspace.** The agent accesses the
   communication region through PCIe memory mapping and handles MMBI, MCTP,
   and PLDM in userspace.
2. **Integration with the BMC userspace MCTP stack.** MMBI support is added to
   libmctp, mctpd, and the related libraries. Host management support is developed
   as an extension of OCP pldmd.
3. **BMC MMBI application ABI compatibility.** The design retains the existing
   PCIe hardware channel. The BMC PCIe MMBI driver provides an application
   interface compatible with the referenced BMC eSPI MMBI interface.
4. **Local validation without the target hardware.** UDS supports local
   MCTP/PLDM/RDE exchanges and BEJ conversion checks. HTTP provides a separate
   way to inspect resource content. These paths operate without a target BMC or
   PCIe MMBI link.

### Scope

The design covers management communication between one host and one BMC while
the host OS is running, together with local resource and protocol validation.
The agent supplements the BMC's out-of-band management capabilities. Management
functions that must remain available when the host is stopped continue to be
provided by the BMC. BIOS-stage protocol access and ownership handover of the
communication channel are separate platform integration topics.

This document defines externally visible capabilities, protocol roles, and
interface compatibility boundaries. Device discovery methods, internal data
models, resource identifier strategies, module structure, caching, and state
management remain implementation choices.

## Proposed Design

### Architecture overview

The host agent exposes FRU, sensor, RDE, and event capabilities over the PCIe
MMBI channel. The BMC userspace MCTP stack provides transport access, while the
OCP pldmd extensions handle PLDM management exchanges. BMC inventory, monitoring,
event, and Redfish services consume the resulting information.

```text
+----------------------------------------------------------------------------+
|  Host OS (userspace)                                                       |
|                                                                            |
|  +-------------------------------+                 +-------------------+   |
|  | Host PLDM Agent               |                 | MCTP / MMBI       |   |
|  | FRU / Sensor / RDE / Event    |<--------------->| userspace stack   |   |
|  +-------------------------------+                 +-------------------+   |
|                                                              ^             |
+--------------------------------------------------------------+-------------+
                                                               |
                                                               v
                                                 +---------------------------+
                                                 | PCIe shared memory        |
                                                 | and notifications         |
                                                 +---------------------------+
                                                               ^
                                                               |
                                                               |
+--------------------------------------------------------------+-------------+
|  BMC                                                         |             |
|                                                              v             |
|  +-------------------+    +-------------------+    +-------------------+   |
|  | OCP pldmd         |<-->| Application MCTP  |<-->| PCIe MMBI driver  |   |
|  | extensions        |    | stack             |    |                   |   |
|  +-------------------+    +-------------------+    +-------------------+   |
|      |           ^                  RDE access                             |
|      |           +-------------------------------------------+             |
|      |                                                       |             |
|      |                                                       |             |
|      v                                                       v             |
|  +-------------------+       Management data       +-------------------+   |
|  | Assets / sensors  |---------------------------->| BMC Redfish       |   |
|  | and events        |                             | service           |   |
|  +-------------------+                             +-------------------+   |
|                                                                            |
+----------------------------------------------------------------------------+
```

Bidirectional connections represent protocol or service interactions.
Unidirectional connections represent the publication and use of management
information. RDE access connects the pldmd extensions to the Redfish service;
inventory, sensor, and event information is integrated through the corresponding
BMC services.

### Host PLDM terminus

The host agent implements a PLDM terminus that communicates with the BMC through
a host-side MCTP endpoint. Host information may be obtained through operating
system interfaces, device management libraries, or other available sources.

For inventory, sensor, and RDE access, the BMC normally acts as the requester
and the host as the responder. When the host initiates an event notification
using a request/response exchange, the host acts as the requester and the BMC
as the event receiver and responder. Requester and responder describe roles
within a transaction.

The host can implement the RDE device role in software, exposing information
available to the host as Redfish resources. The BMC acts as the RDE management
controller (MC), accessing those resources over PLDM and making them available
to management clients through its Redfish service. This division of roles follows
the [RDE specification][10]. BEJ represents resource data in RDE; [libbej][11]
provides an implementation reference for encoding and decoding.

### PCIe MMBI transport

The BMC provides PCIe MMBI support using the ASPEED PCIe hardware capabilities
and the eSPI MMBI interface as references. The host OS provides PCIe device access
and shared memory mapping. The agent maps the communication region and performs
its MMBI and upper-layer protocol processing in userspace.

The BMC PCIe MMBI driver exposes packet transfer and control operations compatible
with the referenced eSPI MMBI application interface. The ABI compatibility target
is the interface between the BMC kernel and BMC applications. The host uses its
own PCIe access interface; communication between the two sides follows MMBI and
its MCTP transport binding.

MCTP over MMBI uses a fixed point-to-point connection. It does not use dynamic
EID assignment or MCTP endpoint discovery. Once the channel is available, the BMC
uses PLDM to discover the host's management capabilities. These transport
properties are defined by the [MCTP over MMBI specification][7].

### BMC MCTP integration

An MMBI binding is added to libmctp to send and receive MCTP packets through the
BMC driver. The MCTP layer handles message fragmentation and reassembly.
mctpd provides complete MCTP messages to management services; mctpwplus and related
libraries connect the PLDM services to the MCTP service.

The MCTP extensions make the PCIe MMBI channel available through the existing
message transport interfaces. The pldmd extensions use those interfaces to access
host management capabilities and receive events. Transport integration and
management protocol handling remain separate responsibilities.

### Extending OCP pldmd

The proposed BMC extensions build on [OCP pldmd][5] and add the following
capabilities for managing the host terminus:

| Extension area | Externally visible capability |
| --- | --- |
| Host discovery and access | Access the MMBI peer through the MCTP service and discover the host's PLDM capabilities |
| FRU and sensor support | Retrieve host inventory and sensor information for the corresponding BMC services |
| RDE requester support | Obtain resource descriptions, dictionaries, and operation results, and provide host resource access to the BMC Redfish service |
| Host event reception | Receive supported PLDM events and initiate the corresponding refresh, logging, or notification actions |
| Management service integration | Expose host information and event results through the appropriate D-Bus, Redfish, or logging interfaces |

For example, a PDR repository change event can indicate that the BMC should
retrieve updated management descriptions. The resulting action depends on the
event's semantics and the BMC's management requirements.

The pldmd RDE extension handles protocol exchanges with the host. The BMC Redfish
service places resources in its externally visible resource tree, processes
management client requests, and enforces access control. The interface between
these components and the placement of resources are BMC integration decisions.

### Standalone development and validation

Local validation exercises the agent's resource and protocol behavior without
a target BMC or PCIe MMBI link. Developers can use data from their Linux host or
predefined test inputs, then move to the target platform for system integration.

The two access paths serve different validation purposes:

- **HTTP resource inspection:** Retrieve Redfish JSON and inspect resource
  properties, collections, relationships, and read results.
- **UDS protocol validation:** Carry MCTP packets over a local socket and use a
  client to exercise PLDM/RDE capability discovery, dictionary retrieval,
  resource access, and BEJ encoding and decoding.

```text
                       +------------------------------+
                       | Host PLDM Agent              |
                       +------------------------------+
                              |                ^
               +--------------+                +--------------+
               |                                              |
               v                                              v
+-----------------------------+                +-----------------------------+
| HTTP resource access        |                | UDS / MCTP packets          |
| Redfish JSON                |                |                             |
+-----------------------------+                +-----------------------------+
               |                                              ^
               |                                              |
               |                                              v
               |                               +-----------------------------+
               |                               | Local PLDM / RDE client     |
               |                               |                             |
               |                               +-----------------------------+
               |                                              |
               |                                              v
               |                               +-----------------------------+
               |                               | bej decoded JSON            |
               |                               |                             |
               |                               +-----------------------------+
               |                                              |
               +--------------+                +--------------+
                              v                v
                       +------------------------------+
                       | Common resource semantics    |
                       +------------------------------+
```

UDS is a local packet transport for development. The protocol client obtains
resources through MCTP, PLDM, and RDE exchanges, while HTTP provides direct
inspection of resource content. This setup exercises the protocol exchanges;
it does not emulate PCIe mapping, MMBI shared buffers, or hardware notifications.

Resources exposed through both paths can be compared under equivalent input
conditions. Expected examples and the applicable Redfish schemas provide
independent checks of properties and relationships. Comparisons account for
sampling time and resource URI placement. The local HTTP interface may also
provide navigation resources that are not exposed through RDE.

Local validation covers resource semantics and protocol behavior. Target-platform
validation covers the PCIe MMBI channel and the integration of BMC management
objects, event handling, and Redfish resource access.

## Alternatives Considered

### A network management service on the host

The host could expose a network management interface for the BMC or a management
platform to consume. This approach can use existing networking tools, but it
requires network reachability, authentication, and service integration.

This proposal uses PLDM to connect host capabilities to the BMC, making use of
the existing platform communication channel and the BMC's external management
interface. The corresponding cost is integrating MMBI, MCTP, and PLDM on both
sides. The local HTTP interface serves development and resource inspection.

### A kernel managed host protocol stack

Implementing MMBI or MCTP processing in the kernel can integrate those functions
with kernel device and resource management. It also ties protocol deployment
more closely to the target kernel environment.

The proposal selects a userspace stack to support independent iteration and
local protocol validation. Deployment still depends on the PCIe access facilities
and permissions available on the target platform, and hardware interoperability
must be evaluated there.

### A private shared memory management protocol

A custom shared memory protocol could directly address a specific information
exchange requirement. It would also require the project to maintain its own data
definitions, capability discovery, and event semantics.

This proposal uses MMBI, MCTP, and PLDM to build on existing specifications and
open-source components. Both sides must implement the selected protocol
capabilities and be evaluated together for compatibility.

## Impacts

### API and compatibility

Compatibility assessment covers three interfaces: the MMBI ABI between the BMC
kernel and applications, the protocol exchanges between host and BMC, and the
interfaces between pldmd and BMC management services. The BMC PCIe MMBI driver
preserves compatibility with the referenced eSPI MMBI application ABI. PLDM
functionality is limited to capabilities supported by both sides.

RDE integration is evaluated across host resources, PLDM/RDE access, and the BMC
Redfish interface to establish that protocol results are represented correctly
in the external management interface.

### System integration

The host requires the agent and the permissions needed to access PCIe resources
and retrieve host information. BMC integration involves the PCIe MMBI driver,
libmctp, mctpd, related interface libraries, the OCP pldmd extensions, and the
connections to Redfish and other management services.

Extending the existing OCP components allows their service infrastructure and
management interfaces to be reused. Integration also needs to establish that
host management capabilities can coexist with the BMC's existing device
management functions.

### Security and access

The BMC Redfish service continues to enforce access control for external
management interfaces. BMC receivers validate resource data, events, and protocol
input supplied by the host. Shared memory access and host data retrieval
permissions are limited to the capabilities required by the deployment.
Access to the development HTTP and UDS interfaces is controlled by the deployment
environment.

### Performance and resource usage

Evaluation covers the response times, host and BMC resource consumption, and
shared channel load caused by FRU reads, sensor access, RDE requests, and event
notifications. Sampling frequency, caching, concurrency, and scheduling remain
implementation choices. Performance targets follow the platform's integration
requirements.

### Developer experience

HTTP resource inspection and UDS protocol access provide separate entry points
for checking resource content and protocol behavior before hardware integration.
Predefined input scenarios and comparable outputs help contributors reproduce
issues, evaluate changes, and participate in interoperability testing.

## Testing

Validation has two levels: local resource and protocol testing, and integration
testing on the target platform. HTTP and UDS support local testing; target
platform testing covers PCIe MMBI communication and BMC management integration.
The criteria below are expressed as observable outcomes.

| Validation area | Expected outcome |
| --- | --- |
| HTTP resource content | Selected resource properties, collections, and relationships match expectations and the applicable Redfish schemas |
| Local MCTP/PLDM | A client can discover supported capabilities and read FRU and sensor data over UDS; unsupported requests receive the appropriate protocol response |
| Local RDE and BEJ | A client can retrieve resources and the required dictionaries, decode BEJ data correctly, and resolve resource relationships |
| HTTP and RDE comparison | Under comparable input conditions, common resources have equivalent semantics and match independently defined expected outputs |
| PLDM events | Selected events complete their defined protocol exchanges and produce observable management results on the BMC |
| PCIe MMBI transport | Host and BMC exchange management messages in both directions, and BMC applications use the channel through the agreed compatible interface |
| BMC management integration | Host information and event results are accessible through the selected BMC management interfaces and agree with the corresponding protocol results |
| Restarts and failures | Communication and management access can recover after host, agent, or BMC restarts, link resets, and communication failures |

Validation results identify the component versions, input scenarios, and expected
outputs so they can support interoperability reviews and regression comparisons.
Hardware access, notifications, and shared memory communication are evaluated
on the target platform. Platform-specific performance criteria follow the
integration requirements.

[1]: https://github.com/AspeedTech-BMC/linux/blob/d0202dc0f7bcd587bfa6a7860334c6d0968df1a8/drivers/soc/aspeed/aspeed-bmc-dev.c "ASPEED BMC PCIe reference"
[2]: https://github.com/AspeedTech-BMC/linux/blob/d0202dc0f7bcd587bfa6a7860334c6d0968df1a8/drivers/soc/aspeed/aspeed-host-bmc-dev.c "ASPEED Host OS PCIe reference"
[3]: https://github.com/ocp-hm-openbmc-opf-ami/meta-core/blob/3ae43b2903acd28f2ebd01ccbe08b70048584019/meta-common/recipes-kernel/linux/linux-onetree/0006-adding-intel-espi-and-lpc-sio-driver.patch "OCP BMC eSPI MMBI reference"
[4]: https://github.com/ocp-hm-openbmc-opf-ami/mctpd/tree/18a51b335cba79a36065ce4533487482e1c7f8f5 "OCP application-level MCTP daemon"
[5]: https://github.com/ocp-hm-openbmc-opf-ami/pldmd/tree/dd3037196944ada8b9f3b9d8a5f37c7cd3d899e4 "OCP PLDM daemon extension base"
[6]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0282_1.0.2.pdf "DSP0282 Memory-Mapped Buffer Interface"
[7]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0284_1.0.1.pdf "DSP0284 MCTP MMBI Transport Binding"
[8]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0248_1.3.0.pdf "DSP0248 PLDM for Platform Monitoring and Control"
[9]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0257_1.0.1_0.pdf "DSP0257 PLDM for FRU Data"
[10]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0218_1.2.0.pdf "DSP0218 PLDM for Redfish Device Enablement"
[11]: https://github.com/openbmc/libbej "OpenBMC libbej"
