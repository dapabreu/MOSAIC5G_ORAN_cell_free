# Docker Images 

In this page the creation of the different used docker images is tackled.

# Initial Notes

By further investigating the docker images referring to the RAN Components (`CUUP`, `CUCP` and the `DU` as well as the `gNB`) we discovered that the FlexRIC container can be easily integrated with any container. The problem that caused this is presented in the Wiki Page **Troubleshooting** in problem **#11**. 

- Firstly start up the `docker-compose` file located in `ci-scripts/yaml_files/5g_rfsimulator_flexric/`
- This compose will automatically start a *named volume* called something like ~5g_rfsimulator_flexric_sharedXappLibs
- Exit the docker-compose and run:

```bash 
# Let's see the docker volumes and copy the 
# id of the volume that holds the config files
docker volume ls 

# This command will run the volume
docker run --rm -it \
  -v flexric_sharedXappLibs:/mnt \
  ubuntu bash

# See the files located in directory, they 
# refer to the SM's used by the FlexRIC
ls /mnt

# Exit the running container and copy the files to hosts currnet directory
docker run --rm -v sharedXappLibs:/data -v $(pwd):/backup ubuntu bash -c "cp -r /data /backup/volume_backup"
```

- These Service Models (SMs) can now be copied into the appropriate directories where you plan to run the flexRIC containers. Just make sure to assign the correct permissions and mount them in the proper path within each container. Example can be seen below: 

```yaml
       volumes:
            - ../../conf_files/gnb-du.sa.band78.106prb.rfsim.conf:/opt/oai-gnb/etc/gnb.conf
            - ./libs/:/usr/local/lib/flexric/
```

And for the FlexRIC, change the image parameters to this:
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

With this, you do not need the extra effort in compiling and building your own docker images, you can simply integrate the FleXRIC directly into the docker-compose file and mount the appropriate folders. Make sure your permissions are **correct**.

# gNB w/ E2 Agent Compatibility   

Even though the OpenAirInterface framework provides a `docker-compose.yml` file responsible for the implementation with a gNB, in a first instance it is better to compile our own gNB and run it together with the just now created FlexRIC. *This approach will be tested later on.*

## gNB Compilation 

Let's `git-clone` the original repository for the gNB. 
```bash
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git
cd openairinterface5g

# to compile the project we use:
cd cmake_targets/

./build_oai -I -w SIMU --gNB --build-e2 --ninja

# This process can take a long time, so don't panic and Ctrl+c out of the running script.
```
| **Parameters**      | **Description**                                                                                           |
|---------------------|-------------------------------------------------------------------------------------------------------|
| `./build_oai`       | Main script to build the OpenAirInterface software.                                    |
| `-I`                | Starts the installation of the necessary dependencies before the build process.                |
| `-w SIMU`           | Specifies the build mode for the simulator (**SIMU**).                                          |
| `--gNB`             | Indicates that the component to be built is the **gNB** (gNodeB).          |
| `--build-e2`        | Activates the build with support to protocol **E2**, used for the communication with the E2 Agent|
| `--ninja`           | Uses the build with **Ninja**, a tool for the management of builds.         |

💡**The usage of the `e2` flag is major, since it will be responsible for creating an E2-Agent compatible gNB**.

## Start the gNB

After compiling the gNB we can start it. Don't forget to install some extra dependencies:

```bash
# in here we can see the gNB files
cd openairinterface5g/cmake_targets/ran_build/build/

# extra dependencies may be needed:
sudo apt-get install build-essential cmake ninja-build libpthread-stubs0-dev libconfig-dev libmnl-dev libsctp-dev iproute2 net-tools iputils-ping

# let's run the gNB (1!)
sudo ./nr-softmodem -O /home/admin/oai/openairinterface5g/ci-scripts/conf_files/gnb.sa.band78.106prb.rfsim.flexric.conf --rfsim -E
```
## Explanation

In **(1!)** we can see the different files used for the gNB startup: 
- `./nr-softmodem`
  - main binary file for the oai-gNB
- `-O /home/admin/oai/openairinterface5g/ci-scripts/conf_files/gnb.sa.band78.106prb.rfsim.flexric.conf`
  - the configuration file specified for the gNB. In our case it is similar to the ones used in the `docker-compose` approach mentioned in latter wiki pages, but with a small addition: 

  ```bash
  e2_agent = {
    near_ric_ip_addr = "192.168.70.150";
    #sm_dir = "/path/where/the/SMs/are/located/"
    sm_dir = "/usr/local/lib/flexric/"
  };
  ``` 
  With this addition, we specify the IP of the Flex-RIC, communicating via *E2AP* protocol. The sm_dir refers to the *Service Models* directory.

- `--rfsim`
  - Activates the radio simulator

- `-E`
  - Activates the E2AP extensions to integrate with a near-RT RIC

## Docker Images Deployment

In order to follow the same trend as the previous implementations, we will not be using the compiled method. In order to remain true to the docker methodology we will have to create a **custom image of gNB** and use the **available docker file available in the flex-ric project** folder to create these two images.

In order to do this, let's firstly create the **docker gnb images**.

## Build the gNB container

For better organization we will be creating some folders firstly:
```bash
mkdir oai && mkdir oai/oai-gnb
mkdir oai/bin && mkdir oai/etc && mkdir oai/flexric
```
Inside the `oai/oai-gnb` folder we create the dockerfile for oai-gnb using `vim`: 

```yml
# Install necessary dependencies
RUN apt-get update && apt-get install -y \
    libpthread-stubs0-dev \
    libconfig-dev \
    libmnl-dev \
    libsctp-dev \
    iproute2 \
    net-tools \
    iputils-ping \
    && apt-get clean

# Copy compiled binaries
COPY ./bin/nr-softmodem /opt/oai-gnb/bin/
COPY ./etc /opt/oai-gnb/etc/

# Copy the Service Models
COPY ./flexric /usr/local/lib/flexric

# Copy the library libparams_libconfig.so
COPY ./bin/libparams_libconfig.so /usr/local/lib/

# Copy the remaining libraries
COPY ./bin/ /usr/local/lib

# Update the LD_LIBRARY_PATH to include /usr/local/lib
ENV LD_LIBRARY_PATH="/usr/local/lib"
```
## Copy build and libraries

We have to copy some files that are necessary for the creation of the oai-gNB. To assure we use a gNB that is E2 compatible we can copy all of the files previously compiled. 

```bash
cp /home/admin/oai/openairinterface5g/cmake_targets/ran_build/build/* /home/admin/oai/oai-gnb/bin

cp -r /home/admin/oai/openairinterface5g/targets/ etc/

cp -r /usr/local/lib/flexric flexric/

mv /home/admin/oai/openairinterface5g/ci-scripts/conf_files/gnb.sa.band78.106prb.rfsim.flexric.conf /home/admin/oai/oai-gnb/etc/

cp gnb.sa.band78.106prb.rfsim.flexric.conf gnb.conf
```

> Some of these may be in different locations in your case.

This should be more or less the structure of the folder: 

```plaintext
.
├── bin/          
├── Dockerfile       
├── etc/            
├── flexric/         
    ├── src/         
    ├── include/     
    ├── examples/    
    └── lib/        
``` 

In order to build the docker container we use `sudo docker build -t oai-gnb:custom .` if you are inside the folder. To run the container use: 

## Run the custom image

```bash 
sudo docker run --rm -it --name oai-gnb \
  --network rfsim5g-oai-public-net \
  -v ~/oai/oai-gnb/etc/gnb.conf:/opt/oai-gnb/etc/gnb.conf \
  -v ~/oai/oai-gnb/flexric:/usr/local/lib/flexric \
  oai-gnb:custom \
  -O /opt/oai-gnb/etc/gnb.conf -E --rfsim
```


# CUUP w/ E2 Agent Compatibility

While compiling the gNB the executables and binaries of the CUUP will be also compiled:

```bash
cd openairinterface5g

# to compile the project we use:
cd cmake_targets/

./build_oai -I --install-optional-packages -w SIMU --gNB --build-e2 --ninja
```
And now, let's start the CUUP in order to see if it has been successful: 

```bash
# in here we can see the CUUP files
cd openairinterface5g/cmake_targets/ran_build/build/

# extra dependencies may be needed:
sudo apt-get install build-essential cmake ninja-build libpthread-stubs0-dev libconfig-dev libmnl-dev libsctp-dev iproute2 net-tools iputils-ping

# let's run the CUUP
sudo ./nr-cuup -O /home/admin/oai/openairinterface5g/ci-scripts/conf_files/gnb-cuup.sa.f1.conf --rfsim -E
```

If it runs, add this to the configuration file (`gnb-cuup.sa.f1.conf`):

```yaml
e2_agent = {
  near_ric_ip_addr = "192.168.70.150";
  #sm_dir = "/path/where/the/SMs/are/located/"
  sm_dir = "/usr/local/lib/flexric/"
};
```

You can also check for messages in the logs containing `e2-agent` using `grep e2` - these can indicate if the CUUP is now compatible with E2 Agents.

Let's now make a folder to contain all of the used build files to run the docker image of `docker-cuup`: 

```bash
mkdir docker-cuup & cd docker-cuup
mkdir bin & mkdir bin/lib
mkdir etc & mkdir etc/targets
mkdir flexric
vim Dockerfile
```
In the dockerfile insert: 

````yaml
FROM ubuntu:24.04

# Install necessary dependencies
RUN apt-get update && apt-get install -y \
    libpthread-stubs0-dev \
    libconfig-dev \
    libmnl-dev \
    libsctp-dev \
    iproute2 \
    net-tools \
    iputils-ping \
    vim \
    && apt-get clean

# Copy compiled binaries
COPY ./bin/nr-cuup /opt/oai-gnb/bin/
COPY ./etc/ /opt/oai-gnb/etc/

# Copy Service Models (SMs)
COPY ./flexric /usr/local/lib/flexric

# Copy libparams_libconfig.so
COPY ./bin/libparams_libconfig.so /usr/local/lib/

# Other libraries
COPY ./bin/lib/ /usr/local/lib

# Update LD_LIBRARY_PATH to include /usr/local/lib
ENV LD_LIBRARY_PATH="/usr/local/lib"

# Define Entrypoint
ENTRYPOINT ["/opt/oai-gnb/bin/nr-cuup"]
````
With this we have a sense of the necessary files to copy:

```bash
cd docker-cuup

cp ~/openairinterface5g/ci-scripts/conf_files/gnb-cucp.sa.f1.conf etc/
cp ~/openairinterface5g/ci-scripts/conf_files/gnb-du.sa.... /etc
cp ~/openairinterface5g/ci-scripts/conf_files/gnb-cu.sa.... /etc

cp -r ~/openairinterface5g/targets/* etc/targets/

cp -r ~/openairinterface5g/cmake_targets/ran_build/build/* bin/

mv * lib

cd bin/

mv lib/libparams_libconfig .
mv lib/librfsimulator.so .
mv lib/nr-cuup .

cd ..

cp -r ~/docker-gnb/flexric/* ~/docker-cuup/flexric

cd ~/docker-cuup
```