# General Purpose

The present Wiki results of the Readme.md available in the main project Folder, where originally, all of this information was contained. The following `readme's` tries to make sense of the numerous projects and subprojects that the current Mosaic5G project has as of today. We will approach the Mosaic5G repository available on Github and Gitlab and the OAI repository available only on Gitlab.

It (more or less) tries to follow, for all the mechanisms and components of the 5G Network (and sometimes 4G LTE), this basic template: 
  1. A small summary of what is the component
  2. The last known update for the project repository
  3. The key elements: not necessarily the sub-projects, but the main features, or main utilities or mechanisms inside the component
  4. Required packages for the instantiation of the component
  5. Supported OS and Kernel (with some recommended hardware settings, if mentioned)
  6. Additional Comments

Some additional implementation tests, problems, and different attempts at implementation are also provided in these Docs.

# Introduction

The mosaic5G project originally was developed in 2015 by Eurocom. The aim of this ecosystem of projects is to implement an E2E (end-to-end) 5G system, emulating all of it's components. In the scientific community, this initiative is regarded as one of the most complete and versatile solutions for open-source emulations in frameworks for network implementation. It since was donated in its entirety to OpenAirInterface. Some components of the Mosaic5G project still reside in the old project repository, while some were transfered to the OpenAirInterface repository.

# Architecture:

**These are the main directories of the project:** 

```jsx
openairinterface5g
├── ci-scripts        : Meta-scripts used by the OSA CI process. Contains also configuration files used day-to-day by CI.
├── cmake_targets     : Build utilities to compile (simulation, emulation and real-time platforms), and generated build files.
├── executables       : Top-level executable source files (gNB, eNB, ...)
├── nfapi             : (n)FAPI code for MAC-PHY interface
├── openair1          : Layer 1 (3GPP LTE Rel-10/12 PHY, NR Rel-15 PHY)
├── openair2          : Layer 2 (3GPP LTE Rel-10 MAC/RLC/PDCP/RRC/X2AP, LTE Rel-14 M2AP, NR Rel-15+ MAC/RLC/PDCP/SDAP/RRC/X2AP/F1AP/E1AP), E2AP
├── openair3          : Layer 3 (3GPP LTE Rel-10 S1AP/GTP, NR Rel-15 NGAP/GTP)
├── radio             : Drivers for various radios such as USRP, AW2S, RFsim, 7.2 FHI, ...
```

- All drivers including **rfsimulator** responsible for data transmission at the Physical layer.
- **OpenairX** - where X is the protocol layer. These files contain definitions, functions, among others for the different protocols used by the emulator.
- **nFAPI** - nFAPI being the interface between the MAC-PHY layer developed by CISCO with 3 different types, depending on the position of the gNB and its *splits*
- **executables** - here are located executable files such as gnb.c, cuup.c, ue.c responsible for initializing the components. When we run a build, like `./nr-uesoftmodem it will load all the libraries, modules, and utilities that were mentioned in these points to be run by the executable.
- **cmake_targets** - here are the builds to generate the files needed to run the executables.

**This type of organization adds to the project:** 

- **Functional separation** - since we have all the layers, protocols, structures, and even drivers, in different functional blocks, which are invoked when necessary.
- “**Simple**” **integration** - as they are organized in blocks, different types of functionalities are easily integrated - e.g., if we do not want to use an algorithm to encrypt the data, we can use another library that contains other algorithms
- **Components have clear** **entrypoints** - all components have their entrypoint inside the `executables` folder.

**Multithreading**:

- Each component runs in separate processes. This gives the OAI project, and its  structures, a different address space. More than that, it makes extensive use of **thread pools** and ITTI (InTer Task Interface) systems.
- **ITTI:** for each protocol or instance, or even interface, we have an associated task. Each task is “awakened” by events, messages or even timed events. [Source](https://openairinterface.org/docs/workshop/3_OAI_Workshop_20170427/training/CN_training_control_plane.pdf?utm_source=chatgpt.com), [Source 2](https://projetos.imd.ufrn.br/reginalab/sdran-openairinterface/-/blob/496fe1726b57e20b5ae98c35356e2139095e537f/openairinterface5g/oai_bkp/common/utils/ocp_itti/itti.md?utm_source=chatgpt.com)
- When this is not the case, as in UE, we have the use of **thread pools,** which are internal threads that operate with specific functions. We continue to have threads according to the ITTI system, but now we have tasks associated with actors that execute them and return the value to the main threads, this encapsulated within this structure. This is justified by the fact that in UE we do not have the complexity of layers that we have in UE, hence it does not depend so much on this system, but is instead more oriented towards linear execution.

**Sockets:** 

- Communication in OAI is done through sockets, to communicate with other network entities. Sockets are managed via **ITTI threads,** dedicated to each stack/protocol. Then different operations are used to ensure non-blocking, socket event detection, etc.
- In addition to sockets, as we know, there is the ITTI system that uses FIFO tasks and structures to communicate, or even pipes.
- Especially between components sockets are almost always used.

```jsx
                                        ┌────────────┐
                                        │   AMF/UPF  │
                                        └────┬───────┘
                                             │
                                ┌────────────┼────────────┐
                                ▼                         ▼
                        ┌─────────────┐           ┌─────────────┐
                        │  SCTP Socket│           │  GTP-U Socket│
                        └─────┬───────┘           └─────┬───────┘
                              │                          │
                              ▼                          ▼
                    ┌────────────────┐        ┌────────────────────┐
                    │ TASK_NGAP (ITTI)│◄─────►│ TASK_GTPV1_U (ITTI) │
                    └────────────────┘        └────────────────────┘
                              ▲                          ▲
                              │                          │
                              ▼                          ▼
                      ┌──────────────────────┐   ┌────────────────────┐
                      │ TASK_RRC (ITTI)      │   │ TASK_PDCP (ITTI)   │
                      └──────────────────────┘   └────────────────────┘
                              ▲                          ▲
                              │                          │
                       (ITTI messages)            (ITTI messages)
                              │                          │
                              ▼                          ▼
                         ┌────────────┐             ┌────────────┐
                         │ ITTI FIFO  │◄────────────│ ITTI FIFO  │
                         └────┬───────┘             └────┬───────┘
                              │                          │
                       ┌──────▼────────┐          ┌──────▼────────┐
                       │  Thread RRC   │          │ Thread GTPU   │
                       └───────────────┘          └───────────────┘

                           [External Communication]
                  [Via sockets + epoll in ITTI threads]

──────────────────────────────────────────────────────────────────────────
                         [UE side – nr-uesoftmodem]

                        ┌──────────────────────────┐
                        │  UE_thread (main loop)   │
                        └─────┬──────────┬─────────┘
                      --------│----------│-------------           
                      | ┌──────▼───┐  ┌───▼────────┐  |
                      | │ DL Task │  │ UL Task     │  | ← thread pools
                      | └─────────┘  └─────────────┘  |
                      |-------------------------------| 
```

# Useful Links

The following work can be attributed to the following links and different articles studied: 

```md
Project Link: https://deiucpt.sharepoint.com/sites/MOSAIC5G
Mosaic5G Home Website: https://mosaic5g.io/
OpenAirInterface Website: https://openairinterface.org/
```

These are the repositories that were useful to this work: 
  - **GitHub** project folder for [mosaic5g](https://github.com/simula/mosaic5g-mosaic5g/tree/master)
  - **Gitlab** project folder for [mosaic5g](https://gitlab.eurecom.fr/mosaic5g)
  - **Gitlab** project folder for [oai](https://gitlab.eurecom.fr/oai/cn5g)
