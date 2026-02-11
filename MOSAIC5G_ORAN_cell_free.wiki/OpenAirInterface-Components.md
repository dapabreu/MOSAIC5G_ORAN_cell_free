# OAI

## Container Definitions:

The `docker-compose.yaml` file of OAI, for the CN can contain the following components:

1. **mysql**:
    - **container_name**: "mysql"
    - **image**: mysql:8.0
    - **volumes**:
      - Create an initial SQL script (`oai_db.sql`) to start the database.
      - Create a healthcheck script (`mysql-healthcheck.sh`) to monitor the health of the MySQL service.
    - **environment**: Sets environment variables such as time zone and database credentials.
    - **healthcheck**: Specifies a command to check the health of the MySQL service.
    - **networks**: Assigns a static IP address to the container in `public_net`.
2. **ims** (IP Multimedia Subsystem):
    - **container_name**: "ims"
    - **image**: oaisoftwarealliance/ims
    - **volumes**: Mounts SIP configuration files and users.
    - **healthcheck**: Uses `pgrep` to check if the `asterisk` process is running.
    - **networks**: Assigns a static IP address to the container in `public_net`.
3. **oai-udr** (Unified Data Repository):
    - **container_name**: "oai-udr"
    - **image**: oaisoftwarealliance/oai-udr
    - ** volumes **: Mounts a configuration file `config.yaml`.
    - **environment**: Defines the time zone.
    - **depends_on**: Specifies dependencies on other containers (mysql, oai-nrf).
    - **networks**: Assigns a static IP address to the container in `public_net`.
4. **oai-udm** (Unified Data Management):
    - Similar to `oai-udr`, but depends on `oai-udr`.
5. **oai-ausf** (Authentication Server Function):
    - Similar to `oai-udm`, but depends on `oai-udm`.
6. **oai-nrf** (Network repository function):
    - Similar to `oai-udr`, but without additional dependencies.
7. **oai-amf** (Access and Mobility Management Function):
    - Similar to `oai-udr`, but depends on several services (`mysql`, `oai-nrf`, `oai-ausf`).
8. **oai-smf** (Session Management Function):
    - Similar to `oai-udr`, but depends on `oai-nrf` and `oai-amf`.
9. **oai-upf** (User plan function):
    - Similar to `oai-udr`, but with additional capabilities (`NET_ADMIN`, `SYS_ADMIN`) and is privileged.
    - Depends on `oai-nrf` and `oai-smf`.
10. **oai-ext-dn** (External Data Network):
    - **privileged**: true, indicates that the container has high privileges.
    - **init**: true, ensures that the init system is used.
    - **container_name**: oai-ext-dn
    - **image**: oaisoftwarealliance/trf-gen-cn5g
    - **entrypoint**: Adds an IP route and keeps the container running.
    - **healthcheck**: Checks if the IP route is configured correctly.
    - **networks**: Assigns a static IP address to the container in `public_net`.
11. **public_net**: Defines a bridge network called `oai-cn5g-public-net` with a specified subnet (`192.168.70.128/26`).


**The definition of each container can be found below:**

- **`mysql:8.0`**
  - It is used to store persistent 5G network data, such as subscriber information and network configuration.
- **`oaisoftwarealliance/oai-amf:v2.0.0`**
  - Access and Mobility Management Function is responsible for managing connections and mobility. This image is used to start the AMF service, which manages the connection of mobile devices to the network and coordinates mobility between different radio cells.
- **`oaisoftwarealliance/oai-smf:v2.0.0`**
  - Session Management Function is responsible for managing sessions and routing packets in the 5G network. This image is used to start the SMF service, which manages data sessions, including creating, modifying, and deleting data sessions.
- **`oaisoftwarealliance/oai-upf:v2.0.0`**
  - User Plane Function is responsible for routing user data on the 5G network. This image is used to start the UPF service, which handles user data traffic, forwarding packets between the radio network and the external network.
- **`oaissoftwarealliance/trf-gen-cn5g:focal`**
  - Contains a traffic generation tool. It is used to simulate user data traffic and test the capacity and performance of the implemented 5G network.
- **`oaisoftwarealliance/oai-gnb:develop`**
  - This image is used to start the gNB service, which implements the 5G radio network base station, managing communication between mobile devices and the core.
- **`oaisoftwarealliance/oai-nr-ue:develop`**
  - This image is used to launch a 5G user device simulator, which simulates a mobile device connecting and communicating with the 5G network.

## Standard Deployment Scheme

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/image-1.png" alt="flexric" width="700">

- **OAI-NR-UE**
  - **Function:** User equipment simulator (UE).
- **OAI-gNB (rfsim)**
  - **Function:** 5G base station simulator (gNB).
- **MySQL Server**
  - **Function:** Database that stores information about users and network configuration.
  - **Connection:** Interacts with OAI-AMF and other components to provide configuration and authentication data.
- **OAI-AMF**
  - **Function:** Access and Mobility Management Function.
  - **Connection:** Connects to MySQL Server, OAI-gNB and OAI-SMF
- **OAI-SMF**
  - **Function:** Session Management Function.
  - **Connection:** Connects to OAI-AMF, OAI-UPF and OAI-NRF.
- **OAI-UPF (OAI-SPGWU)**
  - **Function:** User Plan Function.
  - **Connection:** Connects to OAI-SMF and OAI-EXT-DN to provide internet access.
- **OAI-NRF**
  - **Function:** Network Function Registration Function.
  - **Connection:** Registers other network functions, including AMF and SMF, and provides service discovery.
- **OAI-EXT-DN**
  - **Function:** External Data Network that provides internet connectivity.
  - **Connection:** Connects to the OAI-UPF to forward users' data traffic to the internet.

**💡 In the case of using the *monolithic mode* all gNB functions are integrated into a single entity - that is, the control unit and distributed unit are contained in a single component.**


Additional components (specially regarding docker containers) can be found, but these will be referred to in their specific implementation, in the [Wiki Page for Implementations](https://github.com/Surpr1se0/mosaic5G-docs/wiki/Implementations).