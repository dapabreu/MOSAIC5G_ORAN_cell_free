# Intro

In this wiki page we will install the **FlexRIC** component following different architectures and methodologies. This process encompasses the installation of pre-requisites/dependencies, cloning the repository, code compilation and initial configuration of the system in different methodologies. 

Useful links used in the making of these tutorials:
- [Official Docs for FlexRIC](https://gitlab.eurecom.fr/mosaic5g/flexric/-/blob/dev/README.md#21-test-with-emulators)
- [FlexRIC Integration](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/openair2/E2AP/README.md)
- [docker-compose for FlexRIC](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/ci-scripts/yaml_files/5g_rfsimulator/docker-compose.yaml)
- [docker-compose for 5G full-stack architecture](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/ci-scripts/yaml_files/5g_rfsimulator_flexric/docker-compose.yml)
- [Configuration Files for gNB](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/ci-scripts/conf_files/gnb.sa.band78.106prb.rfsim.flexric.conf)
- [Non-Official Tutorial](https://hackmd.io/@RaffieWinata/SyrHDsL1C?utm_source=preview-mode&utm_medium=rec)
- [Webinar regarding the creation of custom xApps](https://www.youtube.com/watch?time_continue=2287&v=sXeYayAyz1g&embeds_referring_euri=https%3A%2F%2Fopenairinterface.org%2F&source_ve_path=MTM5MTE3LDEzOTExNywxMzkxMTcsMTM5MTE3LDEzOTExNywyODY2Ng)

# FlexRIC Service Implementation

✅ **These instructions follow the release v2.0.0 of FlexRIC. In order to build and setup FlexRIC for later versions please follow these [instructions](FlexRIC-Implementation.md#new-instructions)** - *last updates in 13/05/2025* 

## Pre-Requisites
As per stated in the official **FlexRic** documentation, the pre-requisites are:

- Portainer should be used, not mandatory but highly recommended. Refer to Portainer Implementation in this [wiki page](./Portainer%20Implementation.md).
- Cmake: at least `v3.22`

- SWIG: at least `v4.1`
- gcc: `gcc-10, 12, 13` 
    - > `gcc-10` is not supported

To install these run the following commands. We will start with **gcc**:

```bash
sudo apt update
sudo apt upgrade

sudo apt install build-essential

# This will probably install version 11 - not supported
gcc --version

sudo apt install software-properties-common

# Update the repository
sudo add-apt-repository ppa:ubuntu-toolchain-r/test

sudo apt update

sudo apt install gcc-12 g++-12 

# Let's make the priority of gcc-12 higher. Same applies to gcc-13
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 12 --slave /usr/bin/g++ g++ /usr/bin/g++-12

sudo update-alternatives --config gcc

sudo apt install libmpfr-dev libgmp3-dev libmpc-dev -y

# Check if correct version is gcc-12
gcc --version
```

For the **SWIG** installation we use: 

```bash
# Git-clone the repository
git clone https://github.com/swig/swig.git

# We will need these tools in order to compile SWIG
sudo apt install automake libpcre2-dev bison ibsctp-dev python3 cmake-curses-gui libpcre2-dev python3-dev
 
cd swig/

git checkout release-4.1

./autogen.sh

./configure --prefix=/usr/

sudo make install
```

With all of our dependencies installed we finally can **reboot** the system using `sudo reboot` and move on the FlexRIC installation.

## Installation
```bash
# Install remaining dependencies
sudo apt install python3-dev python3-pip

# Git-clone the project from original sources
git clone https://gitlab.eurecom.fr/mosaic5g/flexric.git 

cd flexric

git checkout release v2.0.0

# if the previously mentioned dependencies are installed this should run without problems. This process can take several minutes
mkdir build && cd build && cmake .. && make -j8 

# This should install all of the services models in the host
sudo make install
```

By default, the SMs are installed in `usr/local/lib/flexric`. The configuration file is located in `usr/local/etc/flexric`. Before testing anything, make sure you are using the correct **IP address** in the **configuration file**. 

```bash 
cd usr/local/etc/flexric

vim flexric.conf
```

Inside the file: 
```bash
[NEAR-RIC]
NEAR_RIC_IP = 127.0.0.1

[XAPP]
DB_DIR = /tmp/
```

In order to test the FlexRIC some scripts are provided by OpenAirInterface. You can run them, using different terminals, and checking the output. 

```bash
./build/examples/ric/nearRT-RIC

./build/examples/emulator/agent/emu_agent_gnb

./build/examples/xApp/c/monitor/xapp_kpm_moni

# Test all of the compiled binaries and files
ctest -j8 --output-on-failure
```

Description of the scripts:

| **Command**                          | **Component**              | **Function**                                                                                  | **Description**                                                                                                       |
|--------------------------------------|-----------------------------|--------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| `./build/examples/ric/nearRT-RIC`    | near-RT RIC                 | Near real-time controller for the RAN                                                     | Manages real-time functions in the network, communicating with E2 Agents and xApps in order to retrieve data e apply policies.       |
| `./build/examples/emulator/agent/emu_agent_gnb` | Agent Emulator (gNB)    | Simulates the behavior of a gNB                                                 | Represents a 5G BS, communicating with the near-RT RIC for testing and development without physical hardware.     |
| `./build/examples/xApp/c/monitor/xapp_kpm_moni` | xApp (KPM Monitor)          | Retrieves and analyzes key performance metrics (KPMs)          | Interacts with the near-RT RIC in order to process metrics sent by the agents, monitoring the performance in the network.     |


Different scripts are used for the different xAPPs, also containing different measurements: 
```bash
# start the KPM monitor xApp
./build/examples/xApp/c/monitor/xapp_kpm_moni

#start the RC monitor xApp - aperiodic subscription for "UE RRC State Change"
./build/examples/xApp/c/monitor/xapp_rc_moni

#start the RC control xApp - RAN control function "QoS flow mapping configuration" (e.g. creating a new DRB)

./build/examples/xApp/c/kpm_rc/xapp_kpm_rc

#start the (MAC + RLC + PDCP + GTP) monitor xApp
./build/examples/xApp/c/monitor/xapp_gtp_mac_rlc_pdcp_moni
```

# FlexRIC container

The FlexRIC Dockerfile is located in `/home/admin/oai/flex-ric/test/docker/ubuntu22`. After selecting this folder, run: 

```bash
docker build -t flexric:latest .

docker run --rm -it --name flexric --network rfsim5g-oai-public-net flexric:latest /bin/bash
```

When inside the container, some aditional steps must be made:

```bash
# change the config, with your prefered IP
vim /usr/local/etc/flexric/flexric.conf

# Run the RIC, as previously stated 
./build/examples/ric/nearRT-RIC

# Don't forget to run some SMs in order to see the FlexRIC in action
```

# Docker-Compose Deployment

For the deployment using a `docker-compose` file, we use the provided one, located in `cd openairinterface5g/ci-scripts/yaml_files/5g_rfsimulator_flexric/`.

This docker file implements 3 elements: `flexric`, `oai-gnb`, and a `oai-nr-ue`: 

In order to run the file use:

```bash
docker-compose up
```

Errors related to the creation of `rf5gsim-public-net` docker-network can ocurr. With some minor changes to the file, the docker runs:

```yaml
networks:
    public_net:
        driver: bridge
        name: oai-cn5g-public-net
        ipam:
            config:
                - subnet: 192.168.70.128/26
        driver_opts:
            com.docker-network.bridge.name: "rfsim5g-public"
```

The different containers will begin to startup. In order to run the different xApps tests one can use: 

```bash
docker exec -it flexric /bin/bash
```

# Custom xApps:

In the **root** directory of the user, create a new `.c` file, named `main`:

```C
// Define the support to the protocol E2AP and SMs
#define E2AP_V1
#define KPM_V3_00

#include "/home/admin/baremetal-rt-ric/flexric/src/xApp/e42_xapp_api.h"
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>


int main(int argc, char *argv[]){
	fr_args_t a = init_fr_args(argc, argv);
        // init xapp
	init_xapp_api(&a);

  usleep(1000);
  
  // get all e2 available nodes
  e2_node_arr_t arr = e2_nodes_xapp_api();
  // if there is at least 1 gNB with RAN Function available
  if(arr.len > 0){
    printf("HELLO WORLD!!!!\n");
    printf("RAN Function ID: %d\n", arr.n[0].ack_rf[0].id);
  }
        // main loop
	while(try_stop_xapp_api() == false)
		usleep(1000);
  // free up memory
  free_e2_node_arr(&arr);

  return 0;
}
```

The **API** of the **xApps** is located in `/home/admin/flexric/src/xApp/e42_xapp_api.h`.

In order to compile follow these instructions:

```plaintex 
> Build the project

> gcc main.c /home/admin/flexric/build/src/xApp/libe42_xapp.a   /usr/local/lib/flexric/*.so -lsctp -o xapp_example

> export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib/flexric

> ./xapp_example
```

# New Instructions


## Bare-metal Setup

In order to install follow: 
- [The old pre-requisites](FlexRIC-Implementation.md#Pre-Requisites)

To setup the project:

```bash
# Install remaining dependencies
sudo apt install python3-dev python3-pip

# Git-clone the project from original sources
git clone https://gitlab.eurecom.fr/mosaic5g/flexric.git 

cd flexric
#THIS STEP CHANGES FROM LAST VERSION
# git checkout release v2.0.0  

# if the previously mentioned dependencies are installed this should run without problems. This process can take several minutes
mkdir build && cd build && cmake .. && make -j8 

# This should install all of the services models in the host
sudo make install
```

Some changes to the `cmake` options can be necessary:

```bash
-DE2AP_V1                     # Activate support for E2AP v1
-DXAPP_MULTILANGUAGE=ON       # add Python support
-DE2AP_VERSION=E2AP_V3        # change the E2AP version
-DKPM_VERSION=KPM_V3_00       # change SM KPM version to be used v3
-DASN1C_EXEC_PATH=/usr/bin/asn1c  # Add the path of the ANS.1 compiler
```

All of the xApps will be available under `/usr/local/flexric/xApp`   
All of the source files for the xApps are available under `/home/admin/baremetal-rt-ric/flexric/examples/xApp/c`
This is a possible example of a configuration file:

```plaintext
[NEAR-RIC]
NEAR_RIC_IP = 192.168.70.150

[XAPP]
DB_PATH = /tmp/flexric/
DB_DIR = /tmp/flexric/
DB_NAME = xapp_db
```

## Docker Image Setup

Go to `flexric\docker` in the main folder of the repository:

Edit the `Dockerfile.flexric.ubuntu` file: 

```cmake
# In the apts section add:
...
       sqlite3 \
       vim \
       tree \
...

cmake -GNinja -DCMAKE_BUILD_TYPE=Release \
      -DE2AP_VERSION=$E2AP_VERSION \
      -DKPM_VERSION=$KPM_VERSION \
      …
      -DXAPP_MULTILANGUAGE=ON \


# After the CMD command:
COPY conf/flexric.conf /usr/local/etc/flexric/flexric.conf
```

Then build the docker image.

```bash
docker build -f docker/Dockerfile.flexric.ubuntu -t oai-flexric .
```

Run the image using: 

```bash 
docker run --rm -it \
  --name oai-flexric \
  --network rfsim5g-oai-e2-net \ # It may be necessary to change this network
  -- ip 192.168.70.150
  -v $(pwd)/conf/flexric.conf:/usr/local/etc/flexric/flexric.conf \
  oai-flexric:dev /bin/bash
```

You can also run every xApp separately:

```bash
docker run --rm -it \
  --name xapp-kpm \
  --network rfsim5g-oai-e2-net \
  --ip 192.168.70.151 \
  oai-flexric:dev \
  /usr/local/flexric/xApp/c/monitor/xapp_kpm_moni
```

When inside the container, consider checking the config file in `/usr/local/etc/flexric/flexric.conf`, changing the values as needed:

```conf
[NEAR-RIC]
NEAR_RIC_IP = 192.168.72.151

[XAPP]
DB_DIR = /tmp/
DB_NAME = xapp_db
```

To initiate the FlexRIC use:

```bash
nearRT-RIC -c /usr/local/etc/flexric/flexric.conf
```

## Docker-Compose Implementation:

After building your custom image, go over to `flexric/docker`. In here we have `docker-compose` file you can run, all together with different emulators (for **gNB** and **UE**).

Example:

```yaml
...
    nearRT-RIC:
        image: oai-flexric:dev
        command: "stdbuf -o0 nearRT-RIC"
        container_name: nearRT-RIC
        networks:
            e2_net:
                ipv4_address: 192.168.72.154
        volumes:
            - ./flexric.conf:/usr/local/etc/flexric/flexric.conf
...
```

## Exploring the data in the DB:

All the data is stored in the `sqlite3` DB automatically by the FlexRIC, in the directory exposed in the `config` file. Run: 

```bash
sqlite3 /tmp/xapp_db

# check the tables created (or available)
.tables 

# Select a table and see entries (limited by X lines)
SELECT * FROM [INSERT-TABLE] LIMIT X;

# Check the columns
PRAGMA table_info(INSERT-TABLE);

# Examples:
SELECT * FROM PDCP_bearer ORDER BY num_retransmit DESC;
SELECT * FROM SLICE WHERE snssai_sst = 1;
```

# Grafana Integration w/ FlexRIC:

Following the creation of the **FlexRIC** docker image, we will be using this custom `docker-compose` file:

```yaml
volumes:
  flexric-db:

services:
  flexric:
    image: oai-flexric:dev
    container_name: oai-flexric
    ports:
      - "36421:36421/sctp"
      - "36422:36422/sctp"
    volumes:
      - flexric-db:/tmp/
    networks:
        rfsim5g-oai-e2-net:
          ipv4_address: 192.168.70.150

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "9006:3000"
    volumes:
      - flexric-db:/tmp/
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    networks:
      grafana-net:
        ipv4_address: 10.1.1.3

networks:
  rfsim5g-oai-e2-net:
    external: true
    name: rfsim5g-oai-e2-net
  grafana-net:
    driver: bridge
    name: grafana-net
    ipam:
      config:
        - subnet: 10.1.1.0/26
```

Install this plugin in grafana in order to connect to the sqlite3 database, executing inside the container: 

```bash
grafana-cli plugins install frser-sqlite-datasource
```

When done, head over to `http://IP:9006` on the browser.

Already inside grafana, head over to: 
```
Settings → Data Sources → Add data source

Select SQLite

Insert: 
/tmp/flexric/xapp_db
file:
mode=ro
```

Then only you can create new dashboards through queries to the database: 

```sql
SELECT
  tstamp / 1000000 AS time,
  dl_id AS slice_id,
  COUNT(DISTINCT rnti) AS ue_count
FROM UE_SLICE
GROUP BY slice_id, time
ORDER BY time DESC
LIMIT 100
```