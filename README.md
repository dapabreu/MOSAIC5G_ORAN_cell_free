# Unlocking E2E Cell-Free Emulation: A Practical Mosaic5G Testbed for O-RAN

This repository hosts the implementation and evaluation framework for an **O-RAN compliant Cell-Free (CF) networking architecture** built upon the **Mosaic5G** ecosystem. 

The core of this project is the integration of User-Centric Cell-Free Massive MIMO concepts within the Open RAN (O-RAN) paradigm. By leveraging the **FlexRIC** controller and **OpenAirInterface (OAI)**, this repository provides the necessary tools to orchestrate distributed Access Points (APs) and implement intelligent Radio Resource Management (RRM) for cell-free deployments.

---

## 📖 Documentation

Detailed documentation, setup guides, and experimental workflows can be found in the [Wiki](https://github.com/dapabreu/MOSAIC5G_ORAN_cell_free/wiki) of this repository.

---

## 🚀 Core Overview

The transition from traditional cellular architectures to **Cell-Free** networks requires a fundamental shift in how User Equipments (UEs) are served. Instead of being tied to a single cell, UEs are served by a coordinated cluster of APs. 

This repository provides:

* **O-RAN Integration:** Implementation of xApps designed for the Near-Real-Time RAN Intelligent Controller (Near-RT RIC) to manage cell-free association logic.
* **Mosaic5G Extension:** Enhancements to the Mosaic5G stack (FlexRIC and OAI) to support the signaling and multi-connectivity requirements of cell-free networking.
* **User-Centric Logic:** Algorithms for dynamic serving cluster formation and power allocation, moving away from the cell-edge limitations of traditional 4G/5G.
* **Evaluation Framework:** Scripts and configurations for simulating or deploying a cell-free O-RAN testbed to validate performance metrics such as throughput, latency, and fairness.

---

## 🏗️ Architecture

The project follows the O-RAN Alliance architecture to ensure modularity and interoperability:

1.  **Service Management and Orchestration (SMO):** For the high-level configuration of the cell-free environment.
2.  **Near-RT RIC (FlexRIC):** Hosting the xApps that perform user-centric cluster formation and interference management.
3.  **E2 Node (OAI GNB/CU/DU):** Modified to support the specific E2 interface messages required for cell-free coordination.

---

## 👥 Contributors

This project was developed through the collaborative efforts of:

* **Francisco Gouveia:** Core developer and researcher responsible for the implementation of the cell-free logic, xApp development, and experimental validation.
* **Marco Silva:** Supervision, architectural design, and research guidance.
* **David Abreu:** Supervision, project coordination, and technical oversight.

---

## 🎓 Acknowledgements

The results and code hosted in this repository are the primary output of the **Master's Thesis of Francisco Gouveia**, conducted under the supervision of **Marco Silva** and **David Abreu**. This work aims to bridge the gap between theoretical Cell-Free Massive MIMO research and practical, open-source O-RAN implementations.

---

> **Note:** For technical inquiries or collaboration, please refer to the contact information of the supervisors or the main contributor via the GitHub profile.
