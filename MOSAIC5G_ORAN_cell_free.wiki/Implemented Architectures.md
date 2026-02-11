# Implementations

This page contains all of the information regarding the configuration, customization and configuration of the OpenAirInterface into several architectures. Additionally, configuration files may be referenced from the main repository of this wiki pages.

# 1. OAI-CN Service Deployment

> Requirements: 4GB memory, 8 processors, 25GB disk (15 used for CN) - [Ubuntu 22.04 LTS](https://releases.ubuntu.com/22.04/ubuntu-22.04.4-desktop-amd64.iso)

## Config Files

Download the configuration files from: `https://gitlab.eurecom.fr/oai/openairinterface5g/-/archive/develop/openairinterface5g-develop.zip?path=doc/tutorial_resources/oai-cn5g.`

This file contains:

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/img.png" alt="flexric" width="700">

Download and extract the files from the OAI repository:

```bash
wget -O ~/oai-cn5g.zip https://gitlab.eurecom.fr/oai/openairinterface5g/-/archive/develop/openairinterface5g-develop.zip?path=doc/tutorial_resources/oai-cn5g
unzip ~/oai-cn5g.zip
mv ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g/doc/tutorial_resources/oai-cn5g ~/oai-cn5g
rm -r ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g ~/oai-cn5g.zip
```

## Starting services

```bash
cd ~/oai-cn5g
docker compose up -d
```

# 2. Full-Stack Docker Implementation

> 4GB of memory, 8 processors, 20GB of disk - [Ubuntu 22.04 LTS](https://releases.ubuntu.com/22.04/ubuntu-22.04.4-desktop-amd64.iso). Implementation of AMF, SMF, UPF = Core. gNB and EU.


## Pull the different Images

This step is optional, as we *up* the different containers they will fetch the corresponding images. Even so, we do:

```bash
$ docker pull mysql:8.0
$ docker pull oaisoftwarealliance/oai-amf:v2.0.0
$ docker pull oaisoftwarealliance/oai-smf:v2.0.0
$ docker pull oaisoftwarealliance/oai-upf:v2.0.0
$ docker pull oaisoftwarealliance/trf-gen-cn5g:focal
$ docker pull oaisoftwarealliance/oai-gnb:develop
$ docker pull oaisoftwarealliance/oai-nr-ue:develop
```

## Core Deployment

```bash
cd ci-scripts/yaml_files/5g_rfsimulator
docker-compose up -d mysql oai-amf oai-smf oai-upf oai-ext-dn
```

By doing this, we are also creating virtual networks that allow communication between different containers. If we do `ifconfig`, we can see that we have two networks - **rfsim5g-traffic & rfsim5g-public:** virtual public network used by containers, and the network used by containers to manage data traffic - this is between the core components for example.

## gNB deployment

**💡 Essentially we are using the **RF Simulator mode here,** this because the radio frequencies, signals and radio channel conditions are being simulated instead of using the same radio hardware (*solves the problem you had of not having the physical equipment .*)**


```bash
docker-compose up -d oai-gnb
```

To check if the gNB connected to the AMF:

```bash
docker logs rfsim5g-oai-amf
```

## NR-UE

```bash
docker-compose up -d oai-nr-ue
```

When performing this deployment, we should check whether the UE is connected:


```bash
docker exec -it rfsim5g-oai-nr-ue /bin/bash
root@bb4d400a832d:/opt/oai-nr-ue# ifconfig

# in the output there must be an interface such as:
oaitun_ue1
```


The `oaitun_ue1` interface is an end-to-end tunnel that goes from the UE to the core network that is in another container.

## Second NR-UE

In the tutorial it is requested that a second entity, or a new UE, be created. In our case, all entities were already created in the `docker-compose.yaml` file. So we just need to do:

```bash
docker-compose up -d oai-nr-ue2

# and check again if you are connected to the core:
docker exec -it rfsim5g-oai-nr-ue2 /bin/bash
root@bb4d400a832d:/opt/oai-nr-ue# ifconfig
```

## Internet connectivity Tests

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/image-2.png" alt="flexric" width="700">


**💡 What we essentially want is to check if our UE can reach the internet through our core network, all through containers.**


```
$ docker exec -it rfsim5g-oai-nr-ue /bin/bash
root@bb4d400a832d:/opt/oai-nr-ue# ping -I oaitun_ue1 -c 10 www.google.com
```

Since this doesn't work, it indicates that I have to modify the DNS values ​​in docker-compose. I couldn't find which values ​​he refers to. For more information about this problem, see the [problems with the implementation.](#problems-with-the-implementations) section.

However, alternatively when using:

```
$ docker exec -it rfsim5g-oai-nr-ue /bin/bash
root@bb4d400a832d# ping -I oaitun_ue1 -c 2 192.168.72.135
```

I was able to run without any problems.

**💡 This test is between the UE and an element outside the core. This proves that we can have communication between the elements of the network.**

## Tests with the iperf server:

*We have to do this test on two different terminals*!

**To connect the iperf server inside the UE container:**

```
docker exec -it rfsim5g-oai-nr-ue /bin/bash
root@bb4d400a832d:/opt/oai-nr-ue# iperf -B 12.1.1.2 -u -i 1 -s
```

**To connect the iperf client inside the ext-dn container:**

```
docker exec -it rfsim5g-oai-ext-dn /bin/bash
root@f239e31a0bd0:/# iperf -c 12.1.1.2 -u -i 1 -t 20 -b 500K
```

With this test, we used `iperf` to prove that there is indeed a data flow coming from the external network to the UE device, in addition to that we were also able to communicate with a second UE, performing a simple ping.

# 3. Full Stack w/FlexRIC

Using the tutorial from the FlexRAN & FlexRIC page, where we build a custom gNB, we will be using that same image in this tutorial, with some minor changes.

The folder structure may have changed a little bit, but this is how it looks, as a reference, for this tutorial:

```plaintext
.docker-gnb/
├── bin/          
├── Dockerfile          # IMPORTANT
├── docker-gnb/
    ├── etc/  
    ├── flexric/    
├── etc/         
    ├── gnb.conf/       # IMPORTANT
    ├── gnb-cu.conf/    # IMPORTANT 
    ├── gnb-du.conf/    # IMPORTANT
    └── targets/        
``` 

We will also use the openairinterface docs, that we have used until now, in order to build the rest of the components - `AMF`, `SMF`, `UPF`, etc...

First of all, we must take in account the IPs that we will use for the different elements and which IP they currently have, in the `oai-rf5gsim-public-net` address space:

| Service      | Container Name        | IP               |
|--------------|--------------------------|------------------|
| MySQL        | rfsim5g-mysql             | 192.168.71.131   |
| OAI AMF     | rfsim5g-oai-amf           | 192.168.71.132   |
| OAI SMF     | rfsim5g-oai-smf           | 192.168.71.133   |
| OAI UPF     | rfsim5g-oai-upf (public)  | 192.168.71.134   |
| OAI UPF     | rfsim5g-oai-upf (traffic) | 192.168.72.134   |
| OAI Ext-DN  | rfsim5g-oai-ext-dn        | 192.168.72.135   |
| OAI gNB     | rfsim5g-oai-gnb           | 192.168.71.140   |
| OAI NR-UE   | rfsim5g-oai-nr-ue         | 192.168.71.150   |


A quick change to the `docker-compose.yml` must also be made. Navigate to `ci-scripts/yaml_files/5g_rfsimulator` and **delete the `depends on:` line** from the UE. The service must have this aspect after the changes:

```yml
      oai-nr-ue:
        image: ${REGISTRY:-oaisoftwarealliance}/${NRUE_IMG:-oai-nr-ue}:${TAG:-develop}
        container_name: rfsim5g-oai-nr-ue
        cap_drop:
            - ALL
        cap_add:
            - NET_ADMIN  # for interface bringup
            - NET_RAW    # for ping
        environment:
            USE_ADDITIONAL_OPTIONS: -E --rfsim -r 106 --numerology 1 --uicc0.imsi 208990100001100 -C 3319680000 --rfsimulator.serveraddr 192.168.71.140 --log_config.global_log_options level,nocolor,time
        networks:
            public_net:
                ipv4_address: 192.168.71.150
        devices:
             - /dev/net/tun:/dev/net/tun
        volumes:
            - ../../conf_files/nrue.uicc.yaml:/opt/oai-nr-ue/etc/nr-ue.yaml
        healthcheck:
            test: /bin/bash -c "pgrep nr-uesoftmodem"
            interval: 10s
            timeout: 5s
            retries: 5
``` 

Next step is to make sure the `amf` and `gnb` have the same values for authentication in the network. Navigate to `5grfsimulator/mini_nonrf_config.yaml` and check for these values: 

```yml
  served_guami_list:
    - mcc: 208
      mnc: 99
      amf_region_id: 01
      amf_set_id: 001
      amf_pointer: 01
  plmn_support_list:
    - mcc: 208
      mnc: 99
      tac: 0x0001
``` 

**💡 They must be the same as the values on the `gnb.conf`.**

From where we can go to the `gnb.conf`, in order to alter the values:

```plaintext

    // Tracking area code, 0x0000 and 0xfffe are reserved values
    tracking_area_code  =  1;

    plmn_list = ({ mcc = 208; mnc = 99; mnc_length = 2; snssaiList = ({ sst = 1; }) });

    nr_cellid = 12345678
```

And also configure the IPs, according to the `docker-compose.yml` file: 

```plaintext
    ////////// AMF parameters:
    amf_ip_address = ({ ipv4 = "192.168.71.132"; });

    NETWORK_INTERFACES :
    {
        GNB_IPV4_ADDRESS_FOR_NG_AMF              = "192.168.71.140";
        GNB_IPV4_ADDRESS_FOR_NGU                 = "192.168.71.140";
        GNB_PORT_FOR_S1U                         = 2152; # Spec 2152
    };
```

Now we can go over to the gNB custom build folder, in our case: `home/admin/docker-gnb`, build the docker image, and start the docker: 

```bash
docker build -t oai-gnb .

sudo docker run --rm -it --name oai-gnb --network rfsim5g-oai-public-net --ip 192.168.71.140 --device /dev/net/tun --cap-add=NET_ADMIN  -v ~/docker-gnb/etc/gnb.conf:/opt/oai-gnb/etc/gnb.conf -v ~/docker-gnb/flexric/flexric:/usr/local/lib/flexric oai-gnb -O /opt/oai-gnb/etc/gnb.conf -E --rfsim --log_config.global_log_options level,nocolor,time
```
Explanation for each parameter/flag is below: 

| Parameter                                      | Description |
|------------------------------------------------|-----------|
| `--rm`                                        | Removes container when stopped. |
| `--name oai-gnb`                              | Naming. |
| `--network rfsim5g-oai-public-net`            | Connects to network `rfsim5g-oai-public-net`. |
| `--ip 192.168.71.140`                         | Gives the container the IP. |
| `--device /dev/net/tun`                       | Allows access for the tunnel device |
| `--cap-add=NET_ADMIN`                         | ADMIN previleges |
| `-v ~/docker-gnb/etc/gnb.conf:/opt/oai-gnb/etc/gnb.conf` | Configuration file mount point |
| `-v ~/docker-gnb/flexric/flexric:/usr/local/lib/flexric` | Directory `flexric` mount point |
| `oai-gnb`                                     | Specifies the image to be used |
| `-O /opt/oai-gnb/etc/gnb.conf`                | Specifies the configuration folder. |
| `--rfsim`                                     | RF Simulator Mode |
| `--log_config.global_log_options level,nocolor,time` | Global options for logs |

**✔ This file can also be found in the projects repository under `gnb-config/gnb.conf`.**

Build the network using the docker-compose.yml from `rfsimulator` folder and see the `amf` logs: 

```bash
docker-compose up -d mysql oai-amf oai-smf oai-upf oai-ext-dn

docker logs rfsim5g-oai-amf
```

It is expected to show up a message of this type: 

```plaintext
$ docker logs rfsim5g-oai-amf
[AMF] [amf_app] [info ] |----------------------------------------------------gNBs' information-------------------------------------------|
[AMF] [amf_app] [info ] |    Index    |      Status      |       Global ID       |       gNB Name       |               PLMN             |
[AMF] [amf_app] [info ] |      1      |    Connected     |         0x0       |         gnb-rfsim        |            208, 99             |
[AMF] [amf_app] [info ] |----------------------------------------------------------------------------------------------------------------|
```

**You have successfully registered the gNB with the CN!**. Now, we need to integrate the FlexRIC. In order, open back up the `gnb.conf` file and add, at the bottom: 

```plaintext
e2_agent = {
  near_ric_ip_addr = "192.168.71.155";
  #sm_dir = "/path/where/the/SMs/are/located/"
  sm_dir = "/usr/local/lib/flexric/"
};
```

Using the same command as before, build back the container, and run. 

Go to the folder `flexric/test/docker/ubuntu22/`, that you have cloned before in the tutorial of FlexRIC and build the FlexRIC image: 

```plaintext
docker build -t flexric:latest .

docker run --rm -it --name flexric --network rfsim5g-oai-public-net --ip 192.168.71.155 flexric:latest /bin/bash	
```

We will start it under the same network as before, using the `--network` and `--ip` flags. Inside the container, you must also change the IP in which the FlexRIC runs, so head to: 

```bash
vim /usr/local/etc/flexric/flexric.conf

# Change the IP to 192.168.71.155

# Run the examples and NearRT RIC 
./build/examples/ric/nearRT-RIC
 
./build/examples/xApp/c/monitor/xapp_kpm_moni
```

Lastly, you can also run the UE, *if you have deleted the `depends on` directive*, using: 

```bash
docker-compose up -d oai-nr-ue
```
-------------------------------
-------------------------------
-------------------------------

# 4. Full Stack w/FlexRIC and CU/DU

In order to run the gNB withou the monolithic mode, OpenAirInterface provides configuration files that can be used without the need for recompiling the gNB with other flags. 

The configuration files are located in `openairinterface5g/ci-scripts/conf_files/gnb-cu.sa.band78.106prb.conf` and `~/openairinterface5g/ci-scripts/conf_files/gnb-du.sa.band78.106prb.rfsim.conf`. 

Different variants of these files are available, but we will be using these.

Before making any changes, copy the configuration files to another location, in this case:

```bash
cp ~/openairinterface5g/ci-scripts/conf_files/gnb-du.sa.band78.106prb.rfsim.conf ~/docker-gnb/etc/gnb-du.conf
```
*Run this command to both of the files - CU and DU*.

We can now make some changes, first start with the **CU**:

```bash
# Don't forget to check if these values match with the AMF.

[...]
plmn_list = ({ mcc = 208; mnc = 99; mnc_length = 2; snssaiList = ({ sst = 1, sd = 0xffffff }) });

nr_cellid = 12345678;

[...]
# Local is the CU
local_s_address = "192.168.71.140";

# The Remote Address is the DU
remote_s_address = "192.168.71.141";
[...]

#
amf_ip_address = ({ ipv4 = "192.168.71.132"; });

NETWORK_INTERFACES :

GNB_IPV4_ADDRESS_FOR_NG_AMF              = "192.168.71.140";
GNB_IPV4_ADDRESS_FOR_NGU                 = "192.168.71.140";
GNB_PORT_FOR_S1U                         = 2152; # Spec 2152

[...]

# Add the e2_agent in order to comunicate with the FlexRIC
e2_agent = {
        near_ric_ip_addr = "192.168.71.155"
        sm_dir="/usr/local/lib/flexric/"
}
```

Now, for the **DU**:

```bash
# These also have to check.

[...]
plmn_list = ({ mcc = 208; mnc = 99; mnc_length = 2; snssaiList = ({ sst = 1, sd = 0xffffff }) });

nr_cellid = 12345678;

[...]
# Local is the DU
local_s_address = "192.168.71.141";

# The Remote Address is the CU
remote_s_address = "192.168.71.140";
[...]

#
amf_ip_address = ({ ipv4 = "192.168.71.132"; });

NETWORK_INTERFACES :

GNB_IPV4_ADDRESS_FOR_NG_AMF              = "192.168.71.140";
GNB_IPV4_ADDRESS_FOR_NGU                 = "192.168.71.140";
GNB_PORT_FOR_S1U                         = 2152; # Spec 2152
```

The only thing missing is now to run both of the containers with different terminals, in order to see the logs: 

**To run the CU:**
```bash 
sudo docker run --rm -it     --name oai-gnb-cu     --network rfsim5g-oai-public-net     --ip 192.168.71.140     --device /dev/net/tun     --cap-add=NET_ADMIN     -v ~/docker-gnb/etc/gnb-cu.conf:/opt/oai-gnb/etc/gnb.conf     -v ~/docker-gnb/flexric/flexric:/usr/local/lib/flexric     oai-gnb     -O /opt/oai-gnb/etc/gnb.conf -E --rfsim --log_config.global_log_options level,nocolor,time
```

**To run the DU:**
```bash 
sudo docker run --rm -it     --name oai-gnb-du     --network rfsim5g-oai-public-net     --ip 192.168.71.141     --device /dev/net/tun     --cap-add=NET_ADMIN     -v ~/docker-gnb/etc/gnb-du.conf:/opt/oai-gnb/etc/gnb.conf     -v ~/docker-gnb/flexric/flexric:/usr/local/lib/flexric     oai-gnb     -O /opt/oai-gnb/etc/gnb.conf -E --rfsim --log_config.global_log_options level,nocolor,time
```

You should see something like this: 

```plaintext 
64574.786349 [UTIL] I threadCreate() for TASK_GTPV1_U: creating thread (no affinity, default priority)
64574.786234 [NR_RRC] I Accepting new CU-UP ID 3584 name gNB-Eurecom-CU (assoc_id -1)
```

Lastly, check the IP from the `UE`, located in the file `~/openairinterface/ci-scripts/yaml_files/5g_rfsimulator/docker-compose` in order to see if it is connected to the right IP address.

It should be pointing to the **DU**, which means - `192.168.71.141`:

```yaml
    oai-nr-ue:
        image: ${REGISTRY:-oaisoftwarealliance}/${NRUE_IMG:-oai-nr-ue}:${TAG:-develop}
        container_name: rfsim5g-oai-nr-ue
        cap_drop:
            - ALL
        cap_add:
            - NET_ADMIN 
            - NET_RAW   
        environment:
            USE_ADDITIONAL_OPTIONS: -E --rfsim -r 106 --numerology 1 --uicc0.imsi 208990100001100 -C 3319680000 --rfsimulator.serveraddr  192.168.71.141 --log_config.global_log_options level,nocolor,time    # IN HERE!!!
```

✔ Now you are ready to run all of the containers and see the AMF register a new **CU**!

Finally, in order to run the `FlexRIC`, use:

```bash
docker run --rm -it --name flexric --network rfsim5g-oai-public-net --ip 192.168.71.155 flexric:latest /bin/bash

# Followed by: 
# change the config, with your prefered IP
vim /usr/local/etc/flexric/flexric.conf

# Run the RIC, as previously stated 
./build/examples/ric/nearRT-RIC

# And run some tests
./build/examples/xApp/c/monitor/xapp_kpm_moni
```

# 5. UE and RAN Disaggregated Setup

In order to continue to use the Docker and a containerized setup of the 5G Network, we will use **Docker Swarms 🐳**. For more info on Swarms, fundamentals and how to setup basic swarms click this [link](https://docs.docker.com/engine/swarm/swarm-tutorial/).

Fundamentally, we will use the **Swarm** to deploy **two** different machines, in the same network (10.3.3.0/16) and then an **overlay network**. This type of `virtual docker network` connects all of the services in the different machines of the swarm and the **swarm manager** assigns random addresses to the containers. 

❌ **This is not the desired behavior, because our 5G components have well-defined IP addresses**

✔ The temporary Fix is to **two overlay networks**, add the containers **manually** through `docker run` always specifying the IP address of such container and the network in which we will be adding the service. This way we are **addressing the services to a fixed IP inside a Swarm**.

First, create a swarm. In our case we will be creating the swarm on the RAN machine: 

```bash
docker swarm init --advertise-addr 10.3.1.244
```

Copy and past the command generated in the second. It should look like this: 

```bash

    docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377
```
If this does not work, maybe there is a change your firewall is blocking the communications. Disable totally with `ufw disable` - only recommended if you are working in private areas or a private network, or disable for the specific ports used by swarms:

```bash
sudo ufw status
sudo ufw allow 2377/tcp
sudo ufw allow 7946/tcp
sudo ufw allow 7946/udp
sudo ufw allow 4789/udp
sudo ufw reload
```

To confirm you have connection maybe telnet into the manager machine from the worker using `telnet [ip] 2377`. 

Run `docker info` to see the current state of the Swarm. You should see 2 nodes. 

After this we need to promote the worker to manager: 

```bash
docker node inspect

# Copy and past the id of the worker

docker node promote [id-of-the-worker]
```

Create both of the networks: 

```bash
docker network create \
  --driver overlay \
  --attachable \
  --subnet=192.168.71.0/24 \	
  rfsim5g-oai-public-net

docker network create \
  --driver overlay \
  --attachable \
  --subnet=192.168.72.0/24 \
  rfsim5g-oai-traffic-net 
```

If you wish to run the UE *in the second machine* as a stack, create a new file called `docker-stack.yml` in the folder of `~openairinterface/ci-scripts/yaml_files/5grfsimulator/`: 

```yaml

services:
  oai-nr-ue:
    image: ${REGISTRY:-oaisoftwarealliance}/${NRUE_IMG:-oai-nr-ue}:${TAG:-develop}
    deploy:
      replicas: 1
    networks:
      - public_net
    cap_add:
      - NET_ADMIN
      - SYS_ADMIN  
    environment:
      USE_ADDITIONAL_OPTIONS: "-E --rfsim -r 106 --numerology 1 --uicc0.imsi 208990100001100 -C 3319680000 --rfsimulator.serveraddr 192.168.71.140 --log_config.global_log_options level,nocolor,time"
    volumes:
      - ../../conf_files/nrue.uicc.yaml:/opt/oai-nr-ue/etc/nr-ue.yaml
    healthcheck:
      test: /bin/bash -c "pgrep nr-uesoftmodem"
      interval: 10s
      timeout: 5s
      retries: 5
    privileged: true

networks:
  public_net:	
    external: true
```

And run: 

```bash
docker stack deploy -c docker-stack.yml oai
```

Alternatively, to run as a docker-compose file: 
```bash
sudo docker-compose up -d oai-nr-ue
```

In the gNB machine, just run the gNB monolithic or CU/DU. **At this time you have all of the components running in 2 different machines.

```bash
sudo docker run --rm -it     --name oai-gnb     --network rfsim5g-oai-public-net     --ip 192.168.71.140     --device /dev/net/tun     --cap-add=NET_ADMIN     -v ~/docker-gnb/etc/gnb.conf:/opt/oai-gnb/etc/gnb.conf     -v ~/docker-gnb/flexric/flexric:/usr/local/lib/flexric     oai-gnb     -O /opt/oai-gnb/etc/gnb.conf -E --rfsim --log_config.global_log_options level,nocolor,time
```



# 6. Full Disaggregated Setup

In order to run a full disaggregated setup, we need 3 different machines. We will be using the same scheme as the previous tutorial but running all the different docker-compose files in the 3 machines.

- `VM01`: Core network and Swarm Manager
- `VM02`: RAN
- `VM03`: UE

**In VM01:**

```bash
docker network create \
  --driver=overlay \
  --attachable \
  --subnet=192.168.71.0/24 \
  --gateway=192.168.71.1 \
  rfsim5g-oai-public-net

docker network create \
  --driver=overlay \
  --attachable \
  --subnet=192.168.72.0/24 \
  --gateway=192.168.72.1 \
  rfsim5g-oai-traffic-net
```

Don't forget to backup the existing file of `docker-compose` before making these changes:

```yaml
# Delete all of the network info, gNB container and UE container and insert this:
networks:
    public_net:
        name: rfsim5g-oai-public-net
        external: true
    traffic_net:
        name: rfsim5g-oai-traffic-net
        external: true
```

After this we can just run: 
```bash
docker compose up -d
```

**In VM02:**
In here, almost no change is necessary. You can run the gNB in CU and DU mode using the commands previously exposed or alternatively you can create a docker compose of those commands. Just make sure you have the `oai-gnb` custom image created.

```yaml
version: '3.8'

services:
  oai-gnb-cu:
    container_name: oai-gnb-cu
    image: oai-gnb
    privileged: true
    devices:
      - "/dev/net/tun"
    cap_add:
      - NET_ADMIN
    volumes:
      - ./gnb-cu.conf:/opt/oai-gnb/etc/gnb.conf
      - ../flexric/flexric/:/usr/local/lib/flexric/
    depends_on:
      - oai-gnb-du
    command: >
      /opt/oai-gnb/bin/nr-softmodem -O /opt/oai-gnb/etc/gnb.conf --sa
    networks:
      rfsim5g-oai-public-net:
        ipv4_address: 192.168.71.140

  oai-gnb-du:
    container_name: oai-gnb-du
    image: oai-gnb
    privileged: true
    devices:
      - "/dev/net/tun"
    cap_add:
      - NET_ADMIN
    volumes:
      - ./gnb-du.conf:/opt/oai-gnb/etc/gnb.conf
      - ../flexric/flexric/:/usr/local/lib/flexric/
    command: >
      /opt/oai-gnb/bin/nr-softmodem -O /opt/oai-gnb/etc/gnb.conf --rfsim --sa -E
    networks:
      rfsim5g-oai-public-net:
        ipv4_address: 192.168.71.141

networks:
    rfsim5g-oai-public-net:
        name: rfsim5g-oai-public-net
        external: true
    rfsim5g-oai-traffic-net:
        name: rfsim5g-oai-traffic-net
        external: true
```

For the monolitic `gNB`:

```yaml
version: '3.8'

services:
  oai-gnb:
    container_name: oai-gnb
    image: oai-gnb
    restart: unless-stopped
    devices:
      - "/dev/net/tun"
    cap_add:
      - NET_ADMIN
    volumes:
      - ./gnb.conf:/opt/oai-gnb/etc/gnb.conf
      - ../flexric/flexric/:/usr/local/lib/flexric/
    command: >
      -O /opt/oai-gnb/etc/gnb.conf -E --rfsim --log_config.global_log_options level,nocolor,time
    healthcheck:
      test: /bin/bash -c "pgrep nr-softmodem"
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      rfsim5g-oai-public-net:
        ipv4_address: 192.168.71.140

networks:
    rfsim5g-oai-public-net:
        name: rfsim5g-oai-public-net
        external: true
    rfsim5g-oai-traffic-net:
        name: rfsim5g-oai-traffic-net
        external: true
```
**✔ Note that these files are also available under the folder `oai-gnb/etc` in the repository files.**


**In VM03:**

In here we will run the `UE`. You can run the component manually, through a stack, or even through docker-compose:

```yaml
services:
    oai-nr-ue:
        image: ${REGISTRY:-oaisoftwarealliance}/${NRUE_IMG:-oai-nr-ue}:${TAG:-develop}
        container_name: rfsim5g-oai-nr-ue
        cap_drop:
            - ALL
        cap_add:
            - NET_ADMIN  # for interface bringup
            - NET_RAW    # for ping
        environment:
            USE_ADDITIONAL_OPTIONS: -E --rfsim -r 106 --numerology 1 --uicc0.imsi 208990100001100 -C 3319680000 --rfsimulator.serveraddr 192.168.71.141 --log_config.global_log_options level,nocolor,time
        networks:
            public_net:
                ipv4_address: 192.168.71.150
        devices:
             - /dev/net/tun:/dev/net/tun
        volumes:
            - ../../conf_files/nrue.uicc.yaml:/opt/oai-nr-ue/etc/nr-ue.yaml
        healthcheck:
            test: /bin/bash -c "pgrep nr-uesoftmodem"
            interval: 10s
            timeout: 5s
            retries: 5
networks:
    public_net:
        name: rfsim5g-oai-public-net
        external: true
```

Additionally, in order to run the FlexRIC, you can choose to have a different machine to run it, or just run it under the RAN machine. In this case we choose to run it in a different machine. All files are available in the `mosaic5G-docs/oai-gnb/flexric`:

```bash
docker build -t flexric:latest .

docker run --rm -it --name flexric --network rfsim5g-oai-public-net --ip 192.168.71.155 flexric:latest /bin/bash	

# Run the examples and NearRT RIC 
./build/examples/ric/nearRT-RIC

# Run the E2 Agents for CU and DU Separately
./build/examples/emulator/agent/emu_agent_gnb_cu
./build/examples/emulator/agent/emu_agent_gnb_du
```

In order to run 2 separate `E2 Agents`, put different ID's in the CU and DU, under the ``gnb-cu.conf`` and ``gnb-du.conf`` files

Or alternatively, run the flexRIC using docker-compose:

Lastly, to run all of this components, manually, you can also run this commands:

- **mysql**

```bash
docker run -d --name rfsim5g-mysql \
  --network rfsim5g-oai-public-net \
  --ip 192.168.71.131 \
  -e TZ=Europe/Paris \
  -e MYSQL_DATABASE=oai_db \
  -e MYSQL_USER=test \
  -e MYSQL_PASSWORD=test \
  -e MYSQL_ROOT_PASSWORD=linux \
  -v $(pwd)/oai_db.sql:/docker-entrypoint-initdb.d/oai_db.sql \
  -v $(pwd)/mysql-healthcheck.sh:/tmp/mysql-healthcheck.sh \
  mysql:8.0
```

- **amf**

```bash
docker run -d --name rfsim5g-mysql \
  --network rfsim5g-oai-public-net \
  --ip 192.168.71.131 \
  -e TZ=Europe/Paris \
  -e MYSQL_DATABASE=oai_db \
  -e MYSQL_USER=test \
  -e MYSQL_PASSWORD=test \
  -e MYSQL_ROOT_PASSWORD=linux \
  -v $(pwd)/oai_db.sql:/docker-entrypoint-initdb.d/oai_db.sql \
  -v $(pwd)/mysql-healthcheck.sh:/tmp/mysql-healthcheck.sh \
  mysql:8.0
```

- **smf**

```bash
docker run -d --name rfsim5g-oai-smf \
  --network rfsim5g-oai-public-net \
  --ip 192.168.71.133 \
  -e TZ=Europe/Paris \
  -v $(pwd)/mini_nonrf_config.yaml:/openair-smf/etc/config.yaml \
  oaisoftwarealliance/oai-smf:v2.0.0
```

- **upf**

```bash
docker run -d --name rfsim5g-oai-upf \
  --network rfsim5g-oai-public-net \
  --ip 192.168.71.134 \
  -e TZ=Europe/Paris \
  -v $(pwd)/mini_nonrf_config.yaml:/openair-upf/etc/config.yaml \
  --cap-add NET_ADMIN --cap-add SYS_ADMIN \
  --privileged \
  oaisoftwarealliance/oai-upf:v2.0.0

docker network connect --ip 192.168.72.134 rfsim5g-oai-traffic-net rfsim5g-oai-upf
```

- **ext-dn**

```bash
docker run -d --name rfsim5g-oai-ext-dn \
  --network rfsim5g-oai-traffic-net \
  --ip 192.168.72.135 \
  --privileged \
  --entrypoint "/bin/bash -c 'iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip route add 12.1.1.0/24 via 192.168.72.134 dev eth0; sleep infinity'" \
  oaisoftwarealliance/trf-gen-cn5g:focal
```

- **UE**

```bash
docker run -d --name rfsim5g-oai-nr-ue \
  --network rfsim5g-oai-public-net\
  --ip 192.168.71.150 \
  --cap-drop ALL \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  --device /dev/net/tun:/dev/net/tun \
  --env USE_ADDITIONAL_OPTIONS="-E --rfsim -r 106 --numerology 1 --uicc0.imsi 208990100001100 -C 3319680000 --rfsimulator.serveraddr 192.168.71.140 --log_config.global_log_options level,nocolor,time" \
  -v $(pwd)/../../conf_files/nrue.uicc.yaml:/opt/oai-nr-ue/etc/nr-ue.yaml \
  --health-cmd "/bin/bash -c 'pgrep nr-uesoftmodem'" \
  --health-interval 10s \
  --health-timeout 5s \
  --health-retries 5 \
  oaisoftwarealliance/oai-nr-ue:develop
```

**gNB**:

```bash
sudo docker run --rm -it     --name oai-gnb     --network rfsim5g-oai-public-net     --ip 192.168.71.140     --device /dev/net/tun     --cap-add=NET_ADMIN     -v ~/docker-gnb/etc/gnb.conf:/opt/oai-gnb/etc/gnb.conf     -v ~/docker-gnb/flexric/flexric:/usr/local/lib/flexric     oai-gnb     -O /opt/oai-gnb/etc/gnb.conf -E --rfsim --log_config.global_log_options level,nocolor,time
```

# 7. Full Disaggregated Setup w/CUCP, CUUP and DU

In this setup we will be deploying these components: 
- Core Network w/ an `AMF`, `SMF` and `UPF`
- RAN w/ 3x `CUUP`, 1x `CUCP`, and 3x `DU`
- 3x `UE`

The `CU-CP `will connect to the `DUs` and `CU-CUs` in order to manage the `UEs`, manage signaling and mobility aspects while making their registration with the `CN` components, establishing the 3x PDUs sessions. 

Use the `docker-compose` file located in `../yaml_files/5g_rfsimulator_e1`.

We will be employing different networks than the last tutorials so go ahead and delete them using `docker network rm *name_of_network*`.

We will be deploying the CNs in `VM01` while the RAN components in `VM02` and the UEs in `VM03`. Create the new networks using the `overlay type` in order to make them available to all of the nodes:

```bash
docker network create --driver overlay --attachable --subnet=192.168.71.128/26 rfsim5g-oai-core-net	

docker network create --driver overlay --attachable --subnet=192.168.72.128/26 rfsim5g-oai-traffic-net

docker network create --driver overlay --attachable --subnet=192.168.72.0/28 rfsim5g-oai-f1c-net

docker network create --driver overlay --attachable --subnet=192.168.73.0/28 rfsim5g-oai-f1u-1-net

docker network create --driver overlay --attachable --subnet=192.168.74.0/28 rfsim5g-oai-f1u-2-net

docker network create --driver overlay --attachable --subnet=192.168.76.0/28 rfsim5g-oai-f1u-3-net

docker network create --driver overlay --attachable --subnet=192.168.77.0/28 rfsim5g-oai-e1-net

docker network create --driver overlay --attachable --subnet=192.168.78.0/28 rfsim5g-oai-ue-net

docker network create --driver overlay --attachable --subnet=192.168.70.0/24 rfsim5g-oai-e2-net
```

Now change the `network` tags in the docker-compose file to match these newly created networks: 

```yaml
networks:
  rfsim5g-oai-core-net:
    name: rfsim5g-oai-core-net
    external: true
  rfsim5g-oai-traffic-net:
    name: rfsim5g-oai-traffic-net
    external: true
  rfsim5g-oai-f1c-net:
    name: rfsim5g-oai-f1c-net
    external: true
  rfsim5g-oai-f1u-1-net:
    name: rfsim5g-oai-f1u-1-net
    external: true
  rfsim5g-oai-f1u-2-net:
    name: rfsim5g-oai-f1u-2-net
    external: true
  rfsim5g-oai-f1u-3-net:
    name: rfsim5g-oai-f1u-3-net
     external: true
  rfsim5g-oai-e1-net:
    name: rfsim5g-oai-e1-net
    external: true
  rfsim5g-oai-ue-net:
    name: rfsim5g-oai-ue-net
    external: true
```

Finally, remove the `cap_drop` directives on all of the containers in order to install debugging packages if needed.

After this, you should be able to deploy the different components in the correct machines:

```bash
docker-compose up -d mysql oai-amf oai-smf oai-upf
docker-compose ps -a
```

Check w/ portainer or `docker ps -a` if the containers are healthy, and then run the RAN components: 

```bash
docker-compose up -d oai-cucp oai-cuup{,2,3} oai-du{,2,3}
```

<details>
<summary>This should be the output of the logs for the CUCP, CUUP and DU, respectively</summary>

```console
----------------CUCP---------------
[...]
18535.139811 [RRC] I Accepting new CU-UP ID 3585 name gNB-OAI (assoc_id 257)
18535.425744 [RRC] I Accepting new CU-UP ID 3584 name gNB-OAI (assoc_id 260)
18535.425757 [RRC] I Accepting new CU-UP ID 3586 name gNB-OAI (assoc_id 261)
18535.669733 [NR_RRC] I Received F1 Setup Request from gNB_DU 3585 (du-rfsim) on assoc_id 263
18535.669814 [RRC] I Accepting DU 3585 (du-rfsim), sending F1 Setup Response
18536.066417 [NR_RRC] I Received F1 Setup Request from gNB_DU 3586 (du-rfsim) on assoc_id 265
18536.066476 [RRC] I Accepting DU 3586 (du-rfsim), sending F1 Setup Response
18536.135581 [NR_RRC] I Received F1 Setup Request from gNB_DU 3584 (du-rfsim) on assoc_id 267
18536.135650 [RRC] I Accepting DU 3584 (du-rfsim), sending F1 Setup Response

----------------CUUP---------------
[...]
122690.500374 [GTPU] I Initializing UDP for local address 192.168.73.2 with port 2153
122690.500406 [GTPU] I Created gtpu instance id: 96
122690.500413 [GTPU] I Configuring GTPu address : 192.168.71.161, port : 2152
122690.500414 [GTPU] I Initializing UDP for local address 192.168.71.161 with port 2152
122690.500420 [GTPU] I Created gtpu instance id: 97

----------------DU---------------
[...]
18626.446953 [NR_MAC] I Frame.Slot 128.0

18629.151076 [NR_MAC] I Frame.Slot 256.0
```
</details>

After, you can finally start up the UEs: 
```bash
docker compose up -d oai-nr-ue{,2,3}
```

<details>
<summary>This should be the output:</summary>

```console
[...]
18758.176149 [NR_RRC] I rrcReconfigurationComplete Encoded 10 bits (2 bytes)
18758.176153 [NR_RRC] I  Logical Channel UL-DCCH (SRB1), Generating RRCReconfigurationComplete (bytes 2)
18758.176154 [NAS] I [UE 0] Received NAS_CONN_ESTABLI_CNF: errCode 1, length 87
18758.176455 [OIP] I Interface oaitun_ue1 successfully configured, ip address 12.1.1.3, mask 255.255.255.0 broadcast address 12.1.1.255
```
</details>

For the final test, attempt to ping the UPF from the UE using the oaitun (tunnel) interface: 

```bash 
docker exec -it rfsim5g-oai-nr-ue /bin/bash
ping -I oaitun_ue1 12.1.1.1
```

# 8. Full Disaggregated Setup w/CUCP, CUUP and DU + FlexRIC

In order to insert the `FlexRIC` into the network, all of the RANs components must be compatible with an E2 Agent. We do not need to compile the custom images made in [Custom Docker Images](Custom-Docker-Images), just refer to the first tutorial shown in that Wiki Page.

Don't forget to the add the flexric directive to the conf.files of each RAN component: 

```conf
e2_agent = {
  near_ric_ip_addr = "192.168.70.150";
  #sm_dir = "/path/where/the/SMs/are/located/"
  sm_dir = "/usr/local/lib/flexric/"
};
```

In order to run all of the components inside a docker-compose, we use the `docker-compose` file used in the previous tutorial, but with slight changes:

- All of the networks must be external
- Insert the flexRIC container in the docker-compose: 
  ```yaml
      oai-flexric:
        image: ${REGISTRY:-oaisoftwarealliance}/oai-flexric:${FLEXRIC_TAG:-develop}
        container_name: oai-flexric
        networks:
            rfsim5g-oai-e2-net:
                ipv4_address: 192.168.70.150
        cap_add:
          - NET_ADMIN
          - SYS_ADMIN
        volumes:
            - ./conf/flexric.conf:/usr/local/etc/flexric/flexric.conf
            - ./libs/:/usr/local/lib/flexric/
        healthcheck:
            test: /bin/bash -c "pgrep nearRT-RIC"
            interval: 10s
            timeout: 5s
  ```
- Mount the SMs to the appropriate directory in the RAN components container: `- ./libs/:/usr/local/lib/flexric/`
- Before running make sure every RAN node has an appropriate ID. See [Additional Notes](Additional-Notes) Chapter for +info.

Now we can simply run: 
```bash
docker-compos up -d oai-flexric
docker-compose up -d oai-cucp oai-cuup oai-du
```
And wait for the FlexRIC to be healthy before running any other containers. The last step is to only separate the containers by using docker-swarm or any other methods as was previously done in the last tutorials.


# 9. Bare Metal Setup w/FlexRIC, gNB and UE

For this implementation the following tutorials/documentation was used: 
  - [Link 1](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed/-/blob/master/docs/DEPLOY_SA5G_BASIC_DEPLOYMENT.md)
  - [Link 2](https://hackmd.io/@praveeng/6GMLAB-OAI-5G-SETUP-GUIDE#Step-3-Setup-OAI-RAN)

We will be using 4 different VMs:
- VM01: Core Network 
- VM02: RAN
- VM03: UE
- VM04: FlexRIC

Firstly, some network settings are important to setup in all of the machines:
```bash
sudo sysctl net.ipv4.conf.all.forwarding=1
sudo sysctl net.ipv4.ip_forward=1

# Disable the firewall only for this specific scenario
sudo ufw disable
sudo iptables -P FORWARD ACCEPT
```

## CN Configuration

Firstly, let's clone the repository with the new images of the Core Network: `git clone https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed .`

Head over to `/home/admin/system-5g/oai-cn5g-fed/docker-compose/docker-compose-basic-nrf.yaml`. 

Before starting make sure your system is updated, has the docker-compose installation and python3 (if not install `python3 python3-pip python3-setuptools`).

We will need to allocate every IP address, so eliminate the comments in the `networks` section of every container. Example can be seen below:

```yaml
networks:
  public_net:
    ipv4_address: 192.168.70.135
```

Also, we will be defining the `demo-oai-public-net` externally so coment that section at the end of the `docker-compose` file:

```yaml
networks:
  public_net:
    external: true
    name: demo-oai-public-net
```

Finally, and optionally also, remove all of `cap_drop` directives from the file as we will sometimes need to install external applications in order to run debug tools and install packages.

In order to create the network use the following command: 
```bash
docker network create \
  --driver=bridge \
  --subnet=192.168.70.128/26 \
  --opt com.docker.network.bridge.name=demo-oai \
  demo-oai-public-net
```
Confirm the network is created with: `docker network ls`

Let's start the docker-compose using the python3 script provided by OAI:

```bash 
python3 core-network.py --type start-basic --scenario 1

# To stop, don't use CTRL + C but instead: 
python3 core-network.py --type stop-basic --scenario 1
```

If the CN is started successfully, `exec -it` into the `EXT-DN` container and ping *dns.google* or even check if the UPF has created a *tun0* interface.

**🧠 Additionally, the gNB must connect to this network using the interface (enX0) of the hosts machine. This means that we will need to add ip route rules to the VM02 (RAN) in order for it to communicate with VM01 (CN).**

IP route to add in VM02: `sudo ip route add 192.168.70.128/26 via 10.3.3.96 dev enX0` 

Attempt to ping any NF from the VM02 specifying the IP Interface: `ping -I enX0 192.168.70.134`

If It does not ping, take a look at the IP Tables Rules in the Core Network VM and see if any docker-implemented rule is currently blocking traffic. It might happen: 

```bash
# see the IP tables rules implemented
sudo iptables -L -n -v
```
If you suspect any rule is blocking traffic just delete the docker implemented rules. Remember every time you run docker-compose the docker-iptables-rules will auto implement themselves so use

```bash
sudo iptables-save > iptables.rules

sudo iptables-restore < iptables.rules
```

> At this moment you should have a CN functioning, up and running...
## RAN Configuration

Clone the repo:

```bash
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git
cd openairinterface5g
git checkout develop 
```

Let's build the gNB and other executables: 

```bash
source oaienv
cd openairinterface5g/cmake_targets/
./build_oai -I -w SIMU --gNB --nrUE --build-e2 --ninja
```

You can also replicate these steps for the UE machine (VM03).

## UE Configuration

- UE Configuration (in progress...)

## FlexRIC Configuration

For the flexRIC Configuration... 


# 10. Thesis Environment

## DU Parameters:

| Field                                            | Description                                              | Value                   |
| ------------------------------------------------ | -------------------------------------------------------- | ----------------------- |
| Active\_gNBs                                     | List of active gNBs                                      | "du-rfsim"              |
| Asn1\_verbosity                                  | ASN.1 verbosity level                                    | none                    |
| gNB\_ID                                          | gNB identifier                                           | 0xe1                    |
| gNB\_DU\_ID                                      | DU identifier                                            | 0xe00                   |
| gNB\_name                                        | gNB name                                                 | "du-rfsim"              |
| tracking\_area\_code                             | Tracking Area Code (TAC)                                 | 1                       |
| plmn\_list                                       | PLMN (MCC/MNC) and SNSSAI slices                         | {208, 99, sst=1,2,3}    |
| nr\_cellid                                       | NR Cell ID                                               | 12345678                |
| pdsch\_AntennaPorts\_XP                          | Number of PDSCH antenna ports                            | 2                       |
| pusch\_AntennaPorts                              | Number of PUSCH antenna ports                            | 2                       |
| do\_CSIRS                                        | Enable CSI-RS transmission                               | 1                       |
| do\_SRS                                          | Enable SRS transmission                                  | 1                       |
| min\_rxtxtime                                    | Minimum RX/TX time                                       | 6                       |
| physCellId                                       | Physical Cell ID                                         | 0                       |
| absoluteFrequencySSB                             | Absolute frequency of SSB                                | 641280                  |
| dl\_frequencyBand                                | Downlink band                                            | 78                      |
| dl\_absoluteFrequencyPointA                      | Absolute Point A DL frequency                            | 640008                  |
| dl\_subcarrierSpacing                            | DL subcarrier spacing (0=15kHz,1=30kHz,2=60kHz,3=120kHz) | 1 (30 kHz)              |
| dl\_carrierBandwidth                             | DL bandwidth (PRBs)                                      | 106                     |
| initialDLBWPlocationAndBandwidth                 | Initial DL BWP location and bandwidth                    | 28875                   |
| initialDLBWPsubcarrierSpacing                    | Initial DL BWP subcarrier spacing                        | 1 (30 kHz)              |
| initialDLBWPcontrolResourceSetZero               | Initial DL BWP control resource set                      | 12                      |
| initialDLBWPsearchSpaceZero                      | Initial DL BWP search space                              | 0                       |
| ul\_frequencyBand                                | Uplink band                                              | 78                      |
| ul\_offstToCarrier                               | UL carrier offset                                        | 0                       |
| ul\_subcarrierSpacing                            | UL subcarrier spacing                                    | 1 (30 kHz)              |
| ul\_carrierBandwidth                             | UL bandwidth (PRBs)                                      | 106                     |
| pMax                                             | Maximum UE power (dBm)                                   | 20                      |
| initialULBWPlocationAndBandwidth                 | Initial UL BWP location and bandwidth                    | 28875                   |
| initialULBWPsubcarrierSpacing                    | Initial UL BWP subcarrier spacing                        | 1 (30 kHz)              |
| prach\_ConfigurationIndex                        | PRACH configuration index                                | 98                      |
| prach\_msg1\_FDM                                 | Number of PRACH Msg1 FDM                                 | 0 (one)                 |
| prach\_msg1\_FrequencyStart                      | PRACH Msg1 frequency start                               | 0                       |
| zeroCorrelationZoneConfig                        | Zero correlation zone                                    | 13                      |
| preambleReceivedTargetPower                      | PRACH target received power                              | -96                     |
| preambleTransMax                                 | Max PRACH transmissions                                  | 6                       |
| powerRampingStep                                 | Power ramping step                                       | 1 (2 dB)                |
| ra\_ResponseWindow                               | Random access response window                            | 4 (8 ms)                |
| ssb\_perRACH\_OccasionAndCB\_PreamblesPerSSB\_PR | CB preambles per SSB (PR)                                | 4                       |
| ssb\_perRACH\_OccasionAndCB\_PreamblesPerSSB     | Preambles per SSB                                        | 14                      |
| ra\_ContentionResolutionTimer                    | Contention resolution timer                              | 7 (64 ms)               |
| rsrp\_ThresholdSSB                               | RSRP threshold for SSB                                   | 19                      |
| prach\_RootSequenceIndex\_PR                     | PRACH root sequence index type                           | 2 (139)                 |
| prach\_RootSequenceIndex                         | PRACH root sequence index                                | 1                       |
| msg1\_SubcarrierSpacing                          | Subcarrier spacing for Msg1                              | 1 (30 kHz)              |
| restrictedSetConfig                              | PRACH restricted set config                              | 0 (unrestricted)        |
| msg3\_DeltaPreamble                              | Msg3 power offset                                        | 1                       |
| p0\_NominalWithGrant                             | UL nominal power with grant                              | -90                     |
| pucchGroupHopping                                | PUCCH hopping mode                                       | 0 (none)                |
| hoppingId                                        | Hopping ID                                               | 40                      |
| p0\_nominal                                      | Nominal PUCCH power                                      | -90                     |
| ssb\_PositionsInBurst\_Bitmap                    | SSB positions in burst bitmap                            | 1                       |
| ssb\_periodicityServingCell                      | SSB periodicity                                          | 2 (20 ms)               |
| dmrs\_TypeA\_Position                            | DMRS Type A position                                     | 0 (pos2)                |
| subcarrierSpacing                                | General subcarrier spacing                               | 1 (30 kHz)              |
| referenceSubcarrierSpacing                       | Reference SCS for TDD                                    | 1 (30 kHz)              |
| dl\_UL\_TransmissionPeriodicity                  | DL/UL transmission periodicity                           | 6 (5 ms)                |
| nrofDownlinkSlots                                | Number of DL slots                                       | 7                       |
| nrofDownlinkSymbols                              | Number of DL symbols                                     | 6                       |
| nrofUplinkSlots                                  | Number of UL slots                                       | 2                       |
| nrofUplinkSymbols                                | Number of UL symbols                                     | 4                       |
| ssPBCH\_BlockPower                               | PBCH block power                                         | 0                       |
| SCTP\_INSTREAMS                                  | SCTP input streams                                       | 2                       |
| SCTP\_OUTSTREAMS                                 | SCTP output streams                                      | 2                       |
| num\_cc (MACRLC)                                 | Number of component carriers                             | 1                       |
| tr\_s\_preference                                | Transport preference MAC–L1                              | local\_L1               |
| tr\_n\_preference (MACRLC)                       | Transport preference MAC–F1                              | f1                      |
| local\_n\_address                                | Local DU address                                         | 192.168.71.171          |
| remote\_n\_address                               | CU address                                               | 192.168.71.150          |
| local\_n\_portc                                  | Local C-plane port                                       | 500                     |
| local\_n\_portd                                  | Local D-plane port                                       | 2153                    |
| remote\_n\_portc                                 | Remote C-plane port                                      | 501                     |
| remote\_n\_portd                                 | Remote D-plane port                                      | 2153                    |
| pusch\_TargetSNRx10                              | Target PUSCH SNR (×10)                                   | 200 (20 dB)             |
| pucch\_TargetSNRx10                              | Target PUCCH SNR (×10)                                   | 200 (20 dB)             |
| num\_cc (L1)                                     | Number of CCs at L1                                      | 1                       |
| tr\_n\_preference (L1)                           | Transport preference L1–MAC                              | local\_mac              |
| prach\_dtx\_threshold                            | PRACH DTX threshold                                      | 200                     |
| pucch0\_dtx\_threshold                           | PUCCH0 DTX threshold                                     | 150                     |
| ofdm\_offset\_divisor                            | OFDM offset divisor                                      | 8                       |
| local\_rf                                        | Use local RF                                             | yes                     |
| nb\_tx                                           | Number of TX antennas                                    | 2                       |
| nb\_rx                                           | Number of RX antennas                                    | 2                       |
| att\_tx                                          | TX attenuation                                           | 0                       |
| att\_rx                                          | RX attenuation                                           | 0                       |
| bands                                            | Supported bands                                          | 78                      |
| max\_pdschReferenceSignalPower                   | Max PDSCH RS power                                       | 20                      |
| max\_rxgain                                      | Max RX gain                                              | 114                     |
| eNB\_instances                                   | Associated eNB instances                                 | \[0]                    |
| clock\_src                                       | Clock source                                             | internal                |
| rfsimulator.serveraddr                           | rfsim server address                                     | "server"                |
| rfsimulator.options                              | rfsim options                                            | "chanmod"               |
| rfsimulator.modelname                            | Channel model name                                       | "Rayleigh1\_corr"       |
| rfsimulator.IQfile                               | IQ trace file                                            | /tmp/rfsimulator.iqs    |
| global\_log\_level                               | Global log level                                         | info                    |
| hw\_log\_level                                   | HW log level                                             | info                    |
| phy\_log\_level                                  | PHY log level                                            | info                    |
| mac\_log\_level                                  | MAC log level                                            | info                    |
| rlc\_log\_level                                  | RLC log level                                            | info                    |
| f1ap\_log\_level                                 | F1AP log level                                           | info                    |
| channelmod.max\_chan                             | Max simulated channels                                   | 12                      |
| channelmod.modellist                             | Channel model list                                       | modellist\_rfsimu\_1    |
| channelmod.\[model]\_type                        | Channel model type                                       | Rayleigh1\_corr         |
| channelmod.ploss\_dB                             | Path loss (dB)                                           | 30                      |
| channelmod.noise\_power\_dB                      | Noise power (dB)                                         | -100                    |
| channelmod.forgetfact                            | Forgetting factor                                        | 0                       |
| channelmod.offset                                | Channel offset                                           | 0                       |
| channelmod.ds\_tdl                               | TDL delay spread                                         | 0                       |
| e2\_agent.near\_ric\_ip\_addr                    | Near-RT RIC IP address                                   | 192.168.70.150          |
| e2\_agent.sm\_dir                                | FlexRIC SM directory                                     | /usr/local/lib/flexric/ |


## Inline parameters used:

```
--rfsim --log_config.global_log_options level,time
--rfsimulator.options chanmod
--gNBs.[0].gNB_ID 0xe40
--gNBs.[0].gNB_DU_ID 0xe0212
--gNBs.[0].nr_cellid 1                          
--gNBs.[0].servingCellConfigCommon.[0].physCellId 1
--MACRLCs.[0].local_n_address 192.168.72.4      
--MACRLCs.[0].remote_n_address 192.168.72.2 
--MACRLCs.[0].local_n_address_f1u 192.168.73.3
--MACRLCs.[0].remote_n_address_f1u 192.168.73.2  
```

## UE Parameters: 

*Note that, IMSIs and Position may vary.*

| Field                            | Description                                              | Value                               |
| -------------------------------- | -------------------------------------------------------- | ----------------------------------- |
| uicc0.imsi                       | IMSI of the UE (subscriber identity)                     | 208990100001100                     |
| uicc0.key                        | Authentication key (K)                                   | fec86ba6eb707ed08905757b1bb44b8f    |
| uicc0.opc                        | Operator Code (OPc)                                      | C42449363BBAD02B66D16BC975D77CC1    |
| uicc0.dnn                        | Default Data Network Name                                | oai                                 |
| uicc0.nssai\_sst                 | Slice/Service Type (SST)                                 | 1                                   |
| position0.x                      | UE position X coordinate                                 | 0.0                                 |
| position0.y                      | UE position Y coordinate                                 | 0.0                                 |
| position0.z                      | UE position Z coordinate (altitude/Earth radius approx.) | 6377900.0                           |
| thread-pool                      | UE thread pool assignment                                | -1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1 |
| channelmod.max\_chan             | Maximum number of simulated channels                     | 10                                  |
| channelmod.modellist             | Channel model list                                       | modellist\_rfsimu\_1                |
| channelmod.model\_name (DL)      | Channel model for DL link                                | rfsimu\_channel\_enB0               |
| channelmod.type (DL)             | DL channel model type                                    | AWGN                                |
| channelmod.ploss\_dB (DL)        | Path loss for DL (dB)                                    | 20                                  |
| channelmod.noise\_power\_dB (DL) | Noise power for DL (dB)                                  | -4                                  |
| channelmod.forgetfact (DL)       | Forgetting factor DL                                     | 0                                   |
| channelmod.offset (DL)           | DL channel offset                                        | 0                                   |
| channelmod.ds\_tdl (DL)          | DL delay spread                                          | 0                                   |
| channelmod                       |                                                          |                                     |


## Inline parameters used:

```
--rfsim --log_config.global_log_options level,nocolor,time
-r 106 --numerology 1 -C 3619200000
--rfsimulator.options chanmod
--uicc0.imsi 208990100001108   
--uicc0.nssai_sst 2  
--ue-nb-ant-rx 2
--ue-nb-ant-tx 2
--rfsimulator.serveraddr 192.168.79.6
```