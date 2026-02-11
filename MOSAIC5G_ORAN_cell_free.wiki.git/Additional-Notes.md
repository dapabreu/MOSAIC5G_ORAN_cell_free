
# Introduction

In this wiki page we explore the different aspects of configuring different network elements like the gNB, UE and more specific NFs.

# UE Configuration Parameters
The UE Configuration Parameters must be set in the 3 different parts of the network. Firstly let's approach the **CN** configuration and provisioning for UEs:

## CN

In the configuration of the **CN** files, located in `./mini_nonrf_config_3slices.yaml` we have these parameters:

```yaml
snssais:
  - &embb_slice1
    sst: 1
  - &embb_slice2
    sst: 1
    sd: 000001 # in hex
  - &custom_slice
    sst: 222
    sd: 00007B # in hex
```
Each slice dictates the type of service and uses a **sd** to differentiate between slices with the same **sst**.

- **Network Slice Selection Assistance Information (NSSAI)**:
  - **sst:** Slice/Service Type
  - **sd:** Slice Differentiator, differentiates slices with the same SST

---

```yaml
served_guami_list:
    mcc: 208
    mnc: 95
    amf_region_id: 01
    amf_set_id: 001
    amf_pointer: 01
```
The Served GUAMI (Globally Unique AMF ID) List identifies the AMF uniquely. It is used so that the UEs know what AMF are using. It contains:

- **MCC/MNC**: identifies the operator
- **Region/Set/Pointer**: identify the AMF inside the operator domain

----
```yaml
plmn_support_list:
  - mcc: 208
    mnc: 95
    tac: 0xa000
    nssai:
      - *embb_slice1
      - *embb_slice2
      - *custom_slice
```

- This portion defines the **PLMNs (Public Land Mobile Networks)** supported, and the slices that the PLMN can use:   
  - **TAC**: Tracking Area Code
  - **nssai**: slice list permited for that specific PLMN

This info is later used in the process of UE Registration so that the AMF knows what slices are available for that specific PLMN.

---
```yaml 
smf_info:
  sNssaiSmfInfoList:
    - sNssai: *embb_slice1
      dnnSmfInfoList:
        - dnn: "oai"
    ...
  - single_nssai: *embb_slice1
    dnn: "oai"
    qos_profile:
      5qi: 9
      session_ambr_ul: "200Mbps"
      session_ambr_dl: "400Mbps"
```

- In here we define:
  - The **DNN**, or Data Network Name
  - The combinations of **slice** + **DNN** for the SMF
  - Each QoS profile associated with each slice and **DNN**. 
----

The steps for all of these parameters to interact are: 
- ``AMF`` verifies the ``GUAMI`` and the `PLMN` that the ``UE`` is trying to connect to
- Associates one or more ``NSSAI`` supported for that ``PLMN``
- The ``UE`` makes the request for a ``PDU`` session (including the ``S-NSSAI`` and ``DNN``)
- The `AMF` consults the `SMF` to verify if that slice + `DNN` is authorized
- The ``SMF`` applies its procedures to establish a PDU Session.

## RAN

The gNB configuration shows:

```yaml
    tracking_area_code  =  0xa000;

    plmn_list = ({ mcc = 208; mnc = 95; mnc_length = 2; snssaiList = ({ sst = 1, sd = 000001 }) });

    nr_cellid = 12345678L
```

Important Notes:
- The `TAC` must match with the one defined in the CN
- In the `PLMN List` it is defined the PLMN that the gNB announces, the slices it supports for that specific PLMN
- The `Cell ID`, that is, a unique identifier of the 5G NR cell.

## UE

The UE configuration normally shows as:

```conf
uicc0 = {
imsi = "208950000000131";
key = "0C0A34601D4F07677303652C0462535B";
opc= "63bfa50ee6523365ff14c1f45f88737d";
dnn= "oai.ipv4";
nssai_sst=1;
nssai_sd=000001;
}
```

- We have to first ensure that the UE is recognized in our Core Network, especially in the database and AMF.
- The operator key, which identifies the AMF, must match the OPC that is in the UE.
- The identification values ​​for each “sim card” are found in: `~/openairinterface5g/ci-scripts/conf_files/nrue.uicc.conf.`
  - IMSI: sim identifier
  - Key
  - OPC
  - DNN: network name
  - NSSAI_SST - type of service slice the UE should use 

In docker-compose.yaml, we can configure these parameters with the following directives, being the volumes parameter, where the file with the UICC configuration (the sim card) is contained.

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/image-3.png" alt="flexric" width="700">

## 2. gNB ID Configuration 

Normally, when using just a **gNB** in its *monolithic* mode, there is no confusion about the **gNB ID**. But when separating the **gNB** into multiple units, as in the case of `CUCP, CUUP and DU`, the IDs and ID assignment process can be a confusing.

Every RAN Node has:
- GNB_ID
- GNB_ID_[COMPONENT_NAME]

The First ID always refers to the higher component of the hierarchy. This means that if we are in the configuration file of a `DU`, the `GNB_ID` is related to the `CUCP`, and if we are in the configuration file of the `CUUP`, the `GNB_ID` is the `CUCP` as well. The other ID, which can be *in this case*, the `GNB_ID_CUUP` or `GNB_ID_DU` always must be different from any other RAN component in order to avoid any problem regarding the identification of the node inside the network. 

An example of assignment of the nodes can be seen in the below table: 

| Component | ID (Hex) | ID (Decimal)
|------------|----------|--------------|
| **CUCP_ID**   | 0xE00    | 3584         |
| **CUUP_ID #1**  | 0xE10    | 3600         |
| **CUUP_ID #1**   | 0xE20    | 3616         |
| **CUUP_ID #1**   | 0xE30    | 3632         |
| **DU_ID #1**   | 0xE11    | 3601         |
| **DU_ID #2**   | 0xE21    | 3617         |
| **DU_ID #3**   | 0xE31    | 3633         |



## 3. Useful commands for Swarms and Swarm Management

```bash
docker stack ps oai
docker service ls
docker network ls
docker node promote [IP]
docker node demote [IP]
sudo docker swarm leave --force
docker info | grep Swarm
```