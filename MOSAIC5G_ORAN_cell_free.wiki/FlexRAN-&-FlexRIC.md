# Intro 

This page contains information comparing both of the implementations made by Mosaic5G/OpenAirInterface in order to emulate a near-RT RIC. The FlexRAN as already been discontinued, now being the FlexRIC the only available option.

# FlexRIC

The FlexRIC is a component of the OpenAirInterface project → with the objective of providing an ecosystem of open source 4G/5G service platforms, towards Open RAN and Open CN: openness and interoperability between equipment from different vendors.

FlexRIC follows the Mosaic5G design principles to be multi-RAT, multi-vendor and multi-service.


💡 Flexible RIC (RAN Inteligent controller) and E2 Agent SDK

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/image-4.png" alt="flexric" width="700">

The RIC is a central component in the O-RAN architecture. It is designed to bring flexibility to the control of the Radio Access Network (RAN). So FlexRIC is designed in such a way that developers and network operators can create custom software-defined RAN controllers.

The E2 Agent SDK is a Software Development Kit designed to facilitate the development of agents that use the E2 interface in the O-RAN architecture: 

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/image-5.png" alt="flexric" width="700">

💡 E2 Interface: standard connection that enables communication between the RIC and the RAN hardware (gNodeBs or eNodeBs).

**Also, the RIC uses this E2 interface to monitor and control the RAN in real-time.**

Additionally, we have the FlexCN, a separate component but with the same principles: SD-CN controller.

## Metrics and Features

Features by component and per-service model: 

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/image-6.png" alt="flexric" width="700">

- KPM (Key Performance Metric): This metric refers to key performance parameters measured in cellular networks, such as throughput, latency, or packet loss.
- RC (Radio Controller): The software component responsible for managing radio resources in the network, including frequency spectrum allocation and transmission power.
- RLC (Radio Link Control): A protocol in the radio control layer that is responsible for organizing reliable data transmission between base stations (BS) and user devices (UE).
- PDCP (Packet Data Convergence Protocol): A protocol in the packet datagram protocol layer responsible for data compression and decompression, as well as providing security functions in mobile networks.
- MAC (Medium Access Control): A protocol in the medium access control layer that regulates access to physical channels in cellular networks, including time and frequency allocation.
- SLICE: This term refers to the concept of network slicing in 5G networks, where the network infrastructure can be divided into virtual chunks that can be organized independently to support different types of services with different qualities.
- TC (Traffic Classifier): A component used to classify network traffic based on certain types or characteristics, such as application or service type.
- GTP (GPRS Tunneling Protocol): A protocol used to transmit data packet traffic between nodes in a mobile network, such as between gateway nodes and base nodes.

**These are some of the RAN monitoring service models (examples):** 

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/image-7.png" alt="flexric" width="700">

*Also, FlexRIC from a memory and storage stanpoint:*

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/image-8.png" alt="flexric" width="400">

- Fine-grained monitoring (thanks to advanced service models)
- Slicing control, traffic control, mobility control
- Network store
- Realtime down to sub ms
- Extendable and plug-and-play service models
- Command line interface application
- Built using C/C++ and Python with support for x64 Linux Systems

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/image-9.png" alt="flexric" width="400">

# FlexRAN

As we already know, SDN or Software Defined Networking is one of the key technologies in 5G and evolving mobile networks. These SDN principals are normally accompanied by: 

- Separation of control plane and data plane via a well defined API
- Support for real-time control (essential to many RAN operations)
- Easily implement new control functions
- Dynamically adapt the control scheme to the network requirements

💡 Essentially, what we look forward is a platform that can perform all these operations

The **FlexRAN** platform, from the Mosaic5G platform is made up of two main components: FlexRAN Control Plane & FlexRAN Agent API. 

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/image-10.png" alt="flexric" width="400">

The FlexRAN control is composed by a **Master Controller** connected to a number of **FlexRAN Agents -** one connected to each eNodeB. The FlexRAN Agent API acts as a southbound API → interface that connects and communicates with components that are lower in the hierarchy. 

*Note: the Master Controller can also be denominated as Real-Time Controller?*

For the communication between the master controller and the agents we have the FlexRAN Protocol.

> *In one direction, the agent sends relevant messages to the master with eNodeB statistics, configurations and events, while in the other direction the master can issue appropriate control commands that define the operation of the agents.*
> 

On top, we have a northbound API which allows RAN Apps to control and modify the state of underlying components such as eNodeBs and UEs based on **statistics and events gathered from the eNodeBs in the FlexRAN control plane.** 

**Flow of operations:** 

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/image-11.png" alt="flexric" width="700">

- **FlexRAN Agent**: This component collects statistics and notifications (protocol messages) from the underlying network or RAN. These messages contain essential data about the network's current state.
- **RIB Updater**: The FlexRAN Agent sends the collected stats and notifications to the RIB Updater. The RIB Updater processes these inputs and generates the necessary updates for the RIB.
- **RIB (Routing Information Base)**: The RIB stores the current state and configuration of the network. The RIB Updater makes RIB update calls to modify the RIB's content based on the latest data from the FlexRAN Agent.
- **Events Notification Service**: This service monitors the RIB and responds to any changes or queries made by applications. It is responsible for notifying applications about significant events or changes in the RIB.
- **Application**: The application queries the RIB to get the current state or configuration of the network. It relies on the Events Notification Service to stay updated on any changes to the RIB that might affect its operation.

RAN control applications can be developed both on the top of the RAN runtime and RTC SDK allowing to monitor, control and coordinate the state of RAN infrastructure. Such applications could vary from soft real-time application including monitoring that obtain statistics reporting to more sophisticated distributed applications that modify the state of the RAN in real-time (e.g. MAC scheduler). All the produced RAN data and APIs are open to be consumed by 3rd parties.

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/image-12.png" alt="flexric" width="400">

- Plug & Play control apps
    - SDK for App-to-App and App-to-RAN
- FlexRAN Master Controller
    - Top level controller/orchestrator
    - eNB/UE state and resources
- FlexRAN Runtime
    - Abstraction and programmability of network functions
    - Extendable RAN APIs
    - Virtualized resources and states
    - Local controller
    - Support for heterogeneous deployments
- FlexRAN Protocol
    - Statistics
    - Configurations
    - Commands
    - Event Trigger
    - Control delegation

## Features

- Control and data plane separation
- Centralized & real-time control: control plane is centralized which simplifies coordination among BS
- Virtualized Control Functions
- Control Delegation & policy reconfiguration: dynamic delegation of control functions and real time reconfiguration of their behavior - also can be implemented using non real time configurations and operations
- User Equipment (UE) Transparency: full transparency to the user equipment ensuring compatibility
- Support of RAN, Content and Device/Provider optimization use cases
- Built using C++ with support for x64 Linux Systems (Master Controller); C for the agent.

**Protocol API:** 

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/image-13.png" alt="flexric" width="700">


## xApps 

This part of the documentation refers to the xApps found in FlexRIC branch `dev`, up to commit no `a358954c12dd009538473dd16554fa62b8835db7`.

### xApps Available:

- xapp_kpm_rc
- xapp_gtp_mac_rlc_pdcp_moni
- xapp_kpm_moni
- xapp_slice_moni
- tc_all
- tc_codel
- tc_enc
- tc_partition
- tc_segregate
- tc_shaper


### xApps Tables

These are the tables generated by the xApps:

- GTP_NGUT
- KPM_MeasRecord
- PDCP_bearer
- SLICE
- KPM_LabelInfo
- MAC_UE
- RLC_bearer
- UE_SLICE

### Metrics

#### GTP_NGUT 
 - Tunnels GTP-U between the gNB & UPF**

| Field           | My Description (old)                         | **According to the Standard**                                                                            |
| --------------- | -------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `tstamp`        | Measurement timestamp.                       | Encoded in the same format as the 64-bit timestamp format as defined in clause 6 of IETF RFC 5905 \[13]. |
| `ngran_node`    | ID of the gNB node (gNB CU or DU).           |                                                                                                          |
| `mcc`, `mnc`    | Mobile Country Code and Mobile Network Code. | Mobile Country Code & Mobile Network Code                                                                |
| `mnc_digit_len` | Number of digits in the MNC (e.g., 2 or 3).  |                                                                                                          |
| `nb_id`         | ID of the base station node (gNB).           |                                                                                                          |
| `cu_du_id`      | Textual identifier of the CU/DU.             |                                                                                                          |
| `teidgnb`       | TEID on the gNB side.                        | Tunnel ID from UPF                                                                                       |
| `rnti`          | UE ID (Radio Network Temporary Identifier).  | This represents a temporary UE Identifier in a cell in a Radio Network.                                  |
| `qfi`           | QoS Flow Identifier.                         | QoS Flow Identifier (QFI) — This IE identifies a QoS flow within a PDU Session.                          |
| `teidupf`       | TEID assigned by the UPF.                    | Tunnel ID from UPF                                                                                       |

#### KPM_MeasRecord
- KPM measures records

| Field            | My Description (old)                         | **According to the Standard**                                                                            |
| ---------------- | -------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `tstamp`         | Measurement timestamp.                       | Encoded in the same format as the 64-bit timestamp format as defined in clause 6 of IETF RFC 5905 \[13]. |
| `ngran_node`     | ID of the gNB node (gNB CU or DU).           |                                                                                                          |
| `mcc`, `mnc`     | Mobile Country Code and Mobile Network Code. | Mobile Country Code & Mobile Network Code                                                                |
| `mnc_digit_len`  | Number of digits in the MNC (e.g., 2 or 3).  |                                                                                                          |
| `nb_id`          | ID of the base station node (gNB).           |                                                                                                          |
| `cu_du_id`       | Textual identifier of the CU/DU.             |                                                                                                          |
| `incompleteFlag` | TEID on the gNB side.                        |                                                                                                          |
| `val`            | UE ID (Radio Network Temporary Identifier).  |                                                                                                          |


#### PDCP_bearer
- Metrics from PDCP Layer

| Field            | My Description (old)                         | **According to the Standard**                                                                                                                       |
| ---------------- | -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tstamp`         | Measurement timestamp.                       | Encoded in the same format as the 64-bit timestamp format as defined in clause 6 of IETF RFC 5905 \[13].                                            |
| `ngran_node`     | ID of the gNB node (gNB CU or DU).           |                                                                                                                                                     |
| `mcc`, `mnc`     | Mobile Country Code and Mobile Network Code. | Mobile Country Code & Mobile Network Code                                                                                                           |
| `mnc_digit_len`  | Number of digits in the MNC (e.g., 2 or 3).  |                                                                                                                                                     |
| `nb_id`          | ID of the base station node (gNB).           |                                                                                                                                                     |
| `cu_du_id`       | Textual identifier of the CU/DU.             |                                                                                                                                                     |
| `txpdu_pkts`     | Transmitted data packet (PDU) metrics.       | Aggregated number of transmitted RLC PDUs                                                                                                           |
| `txpdu_bytes`    | Transmitted data packet (PDU) metrics.       | Aggregated amount of transmitted bytes in RLC PDUs                                                                                                  |
| `txpdu_sn`       | Sequence Number of the last transmitted PDU. | Current sequence number of last TX packet (or TX\_NEXT)                                                                                             |
| `rxpdu_pkts`     | Received data packet (PDU) metrics.          | Aggregated number of received RLC PDUs                                                                                                              |
| `rxpdu_bytes`    | Received data packet (PDU) metrics.          | Amount of bytes received by the RLC                                                                                                                 |
| `rxpdu_sn`       | Sequence Number of the last received PDU.    |                                                                                                                                                     |
| `rxpdu_oo_pkts`  | Out-of-order packets.                        | Aggregated amount of out-of-order RX packets (or RX\_REORD)                                                                                         |
| `rxpdu_oo_bytes` | Out-of-order packets.                        | Aggregated number of out-of-order RX bytes                                                                                                          |
| `rxpdu_dd_pkts`  | Duplicates discarded.                        | Aggregated number of RX packets dropped or discarded by the RLC                                                                                     |
| `rxpdu_dd_bytes` | Duplicates discarded.                        | Aggregated amount of RX bytes dropped or discarded by the RLC                                                                                       |
| `rxpdu_ro_count` | Rollover count of SNs.                       | This state variable indicates the COUNT value following the COUNT value associated with the PDCP Data PDU which triggered t-Reordering. (RX\_REORD) |
| `txsdu_pkts`     | SDU (Service Data Unit) metrics.             | Number of SDUs delivered                                                                                                                            |
| `rxsdu_bytes`    | SDU (Service Data Unit) metrics.             | Number of SDU bytes successfully received so far (counter)                                                                                          |
| `rnti`           | UE ID (Radio Network Temporary Identifier).  | This represents a temporary UE Identifier in a cell in a Radio Network.                                                                             |
| `mode`           | Mode (AM, UM, etc.)                          | 0: RLC AM; 1: RLC UM; 2: RLC TM                                                                                                                     |
| `rbid`           | Radio bearer ID.                             | Radio Bearer Identification — refers to the unique identification assigned to each radio bearer in the network                                      |


#### SLICE
- Info about network slices configured in the node

| Field                        | My Description (old)                                     | **According to the Standard**                                                                            |
| ---------------------------- | -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `tstamp`                     | Measurement timestamp.                                   | Encoded in the same format as the 64-bit timestamp format as defined in clause 6 of IETF RFC 5905 \[13]. |
| `ngran_node`                 | ID of the gNB node (gNB CU or DU).                       |                                                                                                          |
| `mcc`, `mnc`                 | Mobile Country Code and Mobile Network Code.             | Mobile Country Code & Mobile Network Code                                                                |
| `mnc_digit_len`              | Number of digits in the MNC (e.g., 2 or 3).              |                                                                                                          |
| `nb_id`, `cu_du_id`          | ID of the base station node (gNB), or of the CU.         |                                                                                                          |
| `len_slices`                 | Total number of defined slices.                          |                                                                                                          |
| `sched_name`                 | Name of the scheduler in use.                            | Name of available Scheduler                                                                              |
| `id`                         | Slice ID.                                                |                                                                                                          |
| `label`, `type`, `type_conf` | Slice labels and type configuration.                     |                                                                                                          |
| `sched`                      | Scheduling algorithm (e.g., RR, PF).                     |                                                                                                          |
| `type_param*`                | Slice type parameters (e.g., minimum/maximum bandwidth). |                                                                                                          |


#### KPM_LabelInfo
- Labels and metadata associated to KPM Measures

| Field                                                | My Description (old)                                      | **According to the Standard**                                                                                                                                                                                                                                                                          |
| ---------------------------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `tstamp`                                             | Measurement timestamp.                                    |                                                                                                                                                                                                                                                                                                        |
| `ngran_node`                                         | ID of the gNB node (gNB CU or DU).                        | **Global NG-RAN Node ID**<br>This IE is used to globally identify an NG-RAN node in gNB and ng-eNB cases only.                                                                                                                                                                                         |
| `mcc`, `mnc`                                         | Mobile Country Code and Mobile Network Code.              | Mobile Country Code & Mobile Network Code                                                                                                                                                                                                                                                              |
| `mnc_digit_len`                                      | Number of digits in the MNC (e.g., 2 or 3).               |                                                                                                                                                                                                                                                                                                        |
| `nb_id`, `cu_du_id`                                  | ID of the base station node (gNB) or the CU/DU.           | This IE uniquely identifies the gNB-CU-UP at least within a gNB-CU-CP.<br>This IE uniquely identifies the gNB-DU at least within a gNB-CU.                                                                                                                                                             |
| `MeasType`                                           | Type of metric (e.g., throughput, latency).               |                                                                                                                                                                                                                                                                                                        |
| `noLabel`                                            |                                                           | Indicates the associated measurement type without any subcounter.<br>If included, other IEs in 8.3.11 shall not be included in the same Measurement Label (and vice versa).                                                                                                                            |
| `plmnID`, `sST`, `sD`                                | Slice identification.                                     | **PLMN Identity**<br>Represents the SNSSAI subcounter.<br>OCTET STRING of length 1 indicates SST only.<br>OCTET STRING of length 4 indicates SST + SD.<br>OCTET STRING of length 4 with last 3 bytes as `0xFFFFFF` indicates S-NSSAI without SD explicitly.<br>See 3GPP TS 23.003 \[20] clause 28.4.2. |
| `fiveQI`, `qFI`, `qCI`, `aRP*`, `bitrateRange`, etc. | QoS parameters associated with the flow.                  | This IE is used to indicate 5QI value.<br>This IE is used to indicate QCI value.<br>**QoS Flow Identifier (QFI):** identifies a QoS flow within a PDU Session.                                                                                                                                         |
| `layerMU_MIMO`                                       | Number of MIMO layers in use.                             | **Layer at MU-MIMO** — Represents the MIMO layer subcounter.                                                                                                                                                                                                                                           |
| `sUM`                                                |                                                           | **Sum** — Calculated as cumulative sum from the start of the measurement.                                                                                                                                                                                                                              |
| `distBinX`                                           | UE distance to gNB in the X axis.                         | **Distribution Bin X** — An index of Bin X. Only applicable to distribution-type measurement information.                                                                                                                                                                                              |
| `distBinY`                                           | UE distance to gNB in the Y axis.                         | **Distribution Bin Y** — May be present only when Bin X is present.                                                                                                                                                                                                                                    |
| `distBinZ`                                           | UE distance to gNB in the Z axis.                         | **Distribution Bin Z** — May be present only when Bin X and Bin Y are present.                                                                                                                                                                                                                         |
| `preLabelOverride`                                   | Label override indicator from previous processing.        | **Precedent Label Override Indication** — Indicates that subcounters and their values of the previous label apply, except for newly included subcounters.                                                                                                                                              |
| `startEndInd`                                        | Start/end indicator of measurement period.                | Indicates a range of values.<br>When used with Distribution Bin subcounters, it indicates multi-dimensional ranges in ascending order: Bin Z → Bin Y → Bin X.<br>When used otherwise, it indicates a one-dimensional range and the “start” and “end” labels must only differ in that one subcounter.   |
| `min`                                                | Minimum value of the metric observed during the interval. | **Minimum** of the measured values over a granularity period.                                                                                                                                                                                                                                          |
| `max`                                                | Maximum value of the metric observed during the interval. | **Maximum** of the measured values over a granularity period.                                                                                                                                                                                                                                          |
| `avg`                                                | Average value of the metric observed during the interval. | **Average** of the measured values over a granularity period.                                                                                                                                                                                                                                          |

#### MAC_UE
- MAC p/UE metrics

| Field                | My Description (old)                                      | **According to the Standard**                                                                                                   |
| -------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `tstamp`             | Measurement timestamp.                                    | Encoded in the same format as the 64-bit timestamp format as defined in clause 6 of IETF RFC 5905 \[13].                        |
| `ngran_node`         | ID of the gNB node (gNB CU or DU).                        |                                                                                                                                 |
| `mcc`, `mnc`         | Mobile Country Code and Mobile Network Code.              | Mobile Country Code & Mobile Network Code                                                                                       |
| `mnc_digit_len`      | Number of digits in the MNC (e.g., 2 or 3).               |                                                                                                                                 |
| `nb_id`              | ID of the base station node (gNB) or of the CU/DU.        |                                                                                                                                 |
| `cu_du_id`           | Textual identifier of the CU/DU.                          |                                                                                                                                 |
| `frame`              | Frame number where the measurement was taken.             |                                                                                                                                 |
| `slot`               | Slot number within the corresponding frame.               |                                                                                                                                 |
| `dl_aggr_tbs`        | Downlink aggregated transport block size.                 |                                                                                                                                 |
| `ul_aggr_tbs`        | Uplink aggregated transport block size.                   |                                                                                                                                 |
| `dl_aggr_bytes_sdus` | Downlink aggregate bytes of SDUs.                         | Downlink aggregated amount of MAC SDU bytes.                                                                                    |
| `ul_aggr_bytes_sdus` | Uplink aggregate bytes of SDUs.                           | Uplink aggregated amount of MAC SDU bytes.                                                                                      |
| `dl_curr_tbs`        | Current transport block size in downlink.                 |                                                                                                                                 |
| `ul_curr_tbs`        | Current transport block size in uplink.                   |                                                                                                                                 |
| `dl_sched_rb`        | Number of scheduled resource blocks in downlink.          |                                                                                                                                 |
| `ul_sched_rb`        | Number of scheduled resource blocks in uplink.            |                                                                                                                                 |
| `pusch_snr`          | SNR measured on uplink (PUSCH).                           |                                                                                                                                 |
| `pucch_snr`          | SNR measured on uplink (PUCCH).                           |                                                                                                                                 |
| `rnti`               | UE ID (Radio Network Temporary Identifier).               | This represents a temporary UE Identifier in a cell in a Radio Network.                                                         |
| `dl_aggr_prb`        | Total number of PRBs used in DL.                          |                                                                                                                                 |
| `ul_aggr_prb`        | Total number of PRBs used in UL.                          |                                                                                                                                 |
| `dl_aggr_sdus`       | Total number of SDUs transmitted in DL.                   | Downlink aggregate MAC SDU — This measurement provides the transmitted data volume in the downlink during a measurement period. |
| `ul_aggr_sdus`       | Total number of SDUs transmitted in UL.                   | Uplink aggregate MAC SDU — This measurement provides the transmitted data volume in the uplink during a measurement period.     |
| `dl_aggr_retx_prb`   | PRBs used for HARQ retransmissions in DL.                 |                                                                                                                                 |
| `ul_aggr_retx_prb`   | PRBs used for HARQ retransmissions in UL.                 |                                                                                                                                 |
| `wb_cqi`             | Average wideband Channel Quality Indicator.               |                                                                                                                                 |
| `dl_mcs1`            | Modulation and Coding Scheme used in the first DL stream. |                                                                                                                                 |
| `ul_mcs1`            | MCS used in the first UL stream.                          |                                                                                                                                 |
| `dl_mcs2`            | MCS used in the second DL stream (if applicable).         |                                                                                                                                 |
| `ul_mcs2`            | MCS used in the second UL stream (if applicable).         |                                                                                                                                 |
| `phr`                | Power Headroom.                                           |                                                                                                                                 |
| `bsr`                | Buffer Status Report.                                     |                                                                                                                                 |
| `dl_bler`            | Downlink Block Error Rate.                                |                                                                                                                                 |
| `ul_bler`            | Uplink Block Error Rate.                                  |                                                                                                                                 |
| `dl_num_harq`        | Total number of HARQ transmissions in DL.                 |                                                                                                                                 |
| `dl_harq_round0`     | DL transmissions completed on first attempt.              |                                                                                                                                 |
| `dl_harq_round1`     | DL transmissions requiring 1 retransmission.              |                                                                                                                                 |
| `dl_harq_round2`     | DL transmissions requiring 2 retransmissions.             |                                                                                                                                 |
| `dl_harq_round3`     | DL transmissions requiring 3 retransmissions.             |                                                                                                                                 |
| `dlsch_errors`       | Number of DLSCH decoding failures.                        |                                                                                                                                 |
| `ul_num_harq`        | Total number of HARQ transmissions in UL.                 |                                                                                                                                 |
| `ul_harq_round0`     | UL transmissions completed on first attempt.              |                                                                                                                                 |
| `ul_harq_round1`     | UL transmissions with 1 retransmission.                   |                                                                                                                                 |
| `ul_harq_round2`     | UL transmissions with 2 retransmissions.                  |                                                                                                                                 |
| `ul_harq_round3`     | UL transmissions with 3 retransmissions.                  |                                                                                                                                 |
| `ulsch_errors`       | HARQ and retransmission statistics (UL).                  |                                                                                                                                 |


#### RLC_bearer
- Metrics of RLC layer per carrier

| Field                  | My Description (old)                                     | **According to the Standard**                                                                                  |
| ---------------------- | -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `tstamp`               | Measurement timestamp.                                   | Encoded in the same format as the 64-bit timestamp format as defined in clause 6 of IETF RFC 5905 \[13].       |
| `ngran_node`           | ID of the gNB node (gNB CU or DU).                       |                                                                                                                |
| `mcc`, `mnc`           | Mobile Country Code and Mobile Network Code.             | Mobile Country Code & Mobile Network Code                                                                      |
| `mnc_digit_len`        | Number of digits in the MNC (e.g., 2 or 3).              |                                                                                                                |
| `nb_id`                | ID of the base station node (gNB).                       |                                                                                                                |
| `cu_du_id`             | Textual identifier of the CU/DU.                         |                                                                                                                |
| `txpdu_pkts`           | Transmitted data packets (PDUs).                         | Aggregated number of transmitted RLC PDUs                                                                      |
| `txpdu_bytes`          | Transmitted data volume (PDU bytes).                     | Aggregated amount of transmitted bytes in RLC PDUs                                                             |
| `txpdu_wt_ms`          | Average waiting time (ms) before transmission.           | Aggregated head-of-line TX packet waiting time to be transmitted (i.e., sent to the MAC layer)                 |
| `txpdu_dd_pkts`        | Number of PDUs discarded before transmission.            | Number of discarded or dropped PDU packets transmitted                                                         |
| `txpdu_dd_bytes`       | Volume of discarded PDU bytes before transmission.       | Amount of discarded or dropped PDU bytes                                                                       |
| `txpdu_retx_pkts`      | Number of retransmitted PDUs.                            | Number of retransmitted PDU packets                                                                            |
| `txpdu_retx_bytes`     | Volume of retransmitted PDU bytes.                       | Amount of retransmitted PDU bytes                                                                              |
| `txpdu_segmented`      | Number of PDUs segmented for transmission.               | Aggregated number of segmentations                                                                             |
| `txpdu_status_pkts`    | Number of RLC status PDUs transmitted.                   | Aggregated number of TX status PDUs (only applicable to RLC AM)                                                |
| `txpdu_status_bytes`   | Volume of status PDU bytes transmitted.                  | Aggregated amount of TX status bytes (only applicable to RLC AM)                                               |
| `txbuf_occ_bytes`      | Current TX buffer occupancy (bytes).                     | Transmitting bytes currently in buffer                                                                         |
| `txbuf_occ_pkts`       | Number of packets in TX buffer.                          | Current TX buffer occupancy in terms of number of packets                                                      |
| `rxpdu_bytes`          | Total PDU bytes received.                                | Number of received bytes in RLC PDUs                                                                           |
| `rxpdu_pkts`           | Total PDUs received.                                     | Amount of received RLC PDU packets                                                                             |
| `rxpdu_dup_pkts`       | Number of duplicated PDUs received.                      | Aggregated number of duplicated packets                                                                        |
| `rxpdu_dup_bytes`      | Volume of duplicated PDU bytes received.                 | Aggregated number of RX duplicated bytes                                                                       |
| `rxpdu_dd_pkts`        | Number of PDUs discarded after reception.                | Aggregated number of RX packets dropped or discarded by the RLC                                                |
| `rxpdu_dd_bytes`       | Volume of discarded PDU bytes after reception.           | Aggregated amount of RX bytes dropped or discarded by the RLC                                                  |
| `rxpdu_ow_pkts`        | Number of out-of-window PDUs received.                   | Aggregated number of out-of-window received RLC PDUs                                                           |
| `rxpdu_ow_bytes`       | Volume of out-of-window PDU bytes.                       | Aggregated number of out-of-window bytes received in RLC PDUs                                                  |
| `rxpdu_status_pkts`    | Number of status PDUs received.                          | Aggregated number of RX status PDUs (only applicable to RLC AM)                                                |
| `rxpdu_status_bytes`   | Volume of status PDU bytes received.                     | Aggregated number of RX status bytes (only applicable to RLC AM)                                               |
| `rxbuf_occ_bytes`      | Current RX buffer occupancy (bytes).                     | Received bytes currently in buffer                                                                             |
| `rxbuf_occ_pkts`       | Number of packets in RX buffer.                          | Current RX buffer occupancy in terms of number of packets                                                      |
| `txsdu_pkts`           | Number of SDUs delivered for transmission.               | Number of SDU packets delivered                                                                                |
| `txsdu_bytes`          | Volume of transmitted SDU bytes.                         | Number of SDU bytes successfully transmitted so far (counter)                                                  |
| `txsdu_avg_time_to_tx` | Average time from SDU to transmission.                   | 100ms-windowed per packet sojourn time (SDU to PDU) in microseconds                                            |
| `txsdu_wt_us`          | Total waiting time of SDUs in TX buffer (µs).            | Head-of-line (HOL) delay of the current radio bearer, in microseconds                                          |
| `rxsdu_pkts`           | Number of SDUs delivered to upper layer after reception. | Number of SDU packets received                                                                                 |
| `rxsdu_bytes`          | Volume of SDU bytes delivered to upper layer.            | Number of SDU bytes successfully received so far (counter)                                                     |
| `rxsdu_dd_pkts`        | Number of SDUs discarded after reception.                | Number of received discarded or dropped SDU packets                                                            |
| `rxsdu_dd_bytes`       | Volume of SDU bytes discarded.                           | Number of SDU bytes discarded or dropped                                                                       |
| `rnti`                 | UE ID (Radio Network Temporary Identifier).              | This represents a temporary UE Identifier in a cell in a Radio Network.                                        |
| `mode`                 | Mode (AM, UM, etc.).                                     | 0: RLC AM; 1: RLC UM; 2: RLC TM                                                                                |
| `rbid`                 | Radio Bearer ID.                                         | Radio Bearer Identification — refers to the unique identification assigned to each radio bearer in the network |


#### UE_Slice
- UE association to network slices measures

| Field               | My Description (old)                         | **According to the Standard**                                                                            |
| ------------------- | -------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `tstamp`            | Measurement timestamp.                       | Encoded in the same format as the 64-bit timestamp format as defined in clause 6 of IETF RFC 5905 \[13]. |
| `ngran_node`        | ID of the gNB node (gNB CU or DU).           |                                                                                                          |
| `mcc`, `mnc`        | Mobile Country Code and Mobile Network Code. | Mobile Country Code, Mobile Network Code                                                                 |
| `mnc_digit_len`     | Number of digits in the MNC (e.g., 2 or 3).  |                                                                                                          |
| `nb_id`, `cu_du_id` | ID of the base station node (gNB) or the CU. |                                                                                                          |
| `len_ue_slice`      | Number of slices assigned to the UE.         |                                                                                                          |
| `rnti`              | UE ID (Radio Network Temporary Identifier).  | This represents a temporary UE Identifier in a cell in a Radio Network.                                  |
| `dl_id`             | Slice ID in downlink.                        |                                                                                                          |

### Standards Used: 

<aside>
💡 All documentation was done through the standards below, as well as comments found directly in the variable definitions.
</aside>
 

**E2 General Aspects and principles V2**
- O-RAN.WG3.E2GAP-v02.00

**E2 Service Models - Key Performance Metrics**
- E2SM-KPM v2.01
- E2SM-KPM v2.03
- E2SM-KPM v3.00

**E2 Service Models - Ran Control:**
- O-RAN E2 Service Model (E2SM), RAN Control 1.03

**E2 Application Protocol - Technical Specification:** 
- O-RAN.WG3.TS.E2AP-R004-v07.00

**3GPP Reference for the KPM measurement names:**
- TS 32.425
- TS 28.552