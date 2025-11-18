# Introduction to the 5G O-RAN Simulation Lab

Welcome to this hands-on lab where we will build a complete 5G mobile network from scratch using open-source software. The main objective is to explore the **O-RAN (Open Radio Access Network)** architecture. O-RAN is a modern standard that aims to open up mobile networks, which were traditionally closed, single-vendor systems. It achieves this by dividing the base station (the antenna) into smaller components with standard interfaces. This allows network operators to mix and match hardware and software from different manufacturers, fostering innovation. The key division introduced by O-RAN is the **RU-DU-CU** architecture, which separates the functions of the radio access network.

To understand this architecture, let's break down its parts.

1. **The RU (Radio Unit):** This is the layer closest to the user, located in the antenna. Its main function is radio frequency (RF) processing, including analog-to-digital and digital-to-analog signal conversion, amplification, and the transmission/reception of radio waves. It is the direct interface with user devices (UEs).

2. **The DU (Distributed Unit):** It handles real-time and latency-sensitive baseband processing functions, such as encoding/decoding, modulation/demodulation, and parts of the medium access control (MAC). It can be located at a cell site or a nearby data center. The DU connects to multiple RUs.

3. **The CU (Centralized Unit):** It performs upper-layer baseband functions and base station control, which are less latency-sensitive. This includes radio resource control (RRC), admission control, and mobility management functions. The CU connects to multiple DUs and, in turn, to the Core Network.

Every mobile network needs a central brain to manage users, security, and internet connectivity; this is called the **Core Network (or 5GC in 5G)**. In our lab, we'll be using **Open5GS**, an open-source project that implements a complete 5G Core. We chose it for its ease of use, stability, and because it's designed to run seamlessly in containers, simplifying deployment. The Core authorizes the user's device, known as the **User Equipment** (UE), which essentially authorizes your mobile phone to join the network.

To simulate the radio network (the RU-DU-CU part), we'll be using **srsRAN**. This is an amazing open-source software project that implements the entire 4G and 5G radio protocol stack. In our exercise, one srsRAN component will act as a **gNB** (the 5G base station), combining the functions of the **CU and DU** into a single program. Another srsRAN component will simulate the **UE** (the phone). To simulate the **RU** and the air link, srsRAN uses an ingenious *driver*, ZMQ, which replaces the radio frequency with a local network connection between our programs.

To build this complex environment on a single machine, we rely on **Docker**. This "container" technology allows us to package each component (the Core, the gNB, and the UE) into isolated software boxes. These boxes, or containers, are much lighter than virtual machines and ensure that one software's dependencies don't conflict with another's. All of this will run on our **host machine**, which is simply our physical computer (running a Linux operating system) that hosts the Docker software and all the containers that make up the network.

Once the network is up and running, we need to measure it. To do this, we'll use two essential tools. **Wireshark** is a "protocol analyzer" or *sniffer*, which allows us to capture and view every data packet flowing through the network; This is vital for debugging and understanding what the components are "talking" to each other. To generate traffic and measure performance, we'll use iperf3, a standard tool that acts as a speedometer to measure the maximum bandwidth our simulated network can handle.

The ultimate goal of these measurements is to evaluate performance and quality. We'll perform latency tests (also known as delay), which measures how long it takes for a packet to travel from the UE to the core and back; this is critical for interactive applications like online games. We'll also analyze QoS (Quality of Service), which is the network's ability to prioritize traffic. For example, a video call (sensitive to delay) should take priority over downloading a large file. Simulating and measuring this is fundamental to understanding how O-RAN architectures can deliver on the stringent performance promises of 5G.

## Prerequisites

Before starting the lab build, ensure you have the following environment prepared on your host machine:

* **Operating System:** A host machine running **Linux**. Ubuntu 20.04 LTS or 22.04 LTS is recommended for optimal compatibility.

* **Container Software:** **Docker** and **Docker Compose** must be installed and working. This is essential for managing the different network components in isolated environments. [Official installation guide.](https://docs.docker.com/engine/install)

* **Version Control:** **Git** is required to clone the Open5GS and srsRAN software repositories. [Official Installation Guide.](https://git-scm.com/install/linux)
* **Building Tools:** You will need basic building tools (such as `build-essential` and `cmake` on Ubuntu) to compile srsRAN from source and build the Docker images.

```sudo apt-get install cmake make gcc g++ build-essential pkg-config libfftw3-dev libmbedtls-dev libsctp-dev libyaml-cpp-dev libgtest-dev```

* **Basic Knowledge:** Familiarity with the **Linux terminal** (basic commands) and fundamental concepts of **IP networking** (addressing, subnets) is assumed.

## O-RAN Architecture

For this exercise, we will not build a complete O-RAN network with F1/E1/O1 interfaces, as our focus is on simulation and performance evaluation in a controlled environment. Instead, we will emulate a complete 5G architecture where srsRAN and Open5GS interact within containers.

Our architecture will simulate the O-RAN partitioning as follows:

* **Core (5GC):** We will use **Open5GS** (in a container). This project implements all the components of the 5G Core (AMF, SMF, UPF, etc.) and will act as the centralized "brain" of our network.

* **CU + DU (Centralized and Distributed Unit):** The **gNB (5G Base Station) of srsRAN** (in a container) will act as a monolithic unit that combines the functions of the **CU and DU**. It will connect to the Open5GS Core via the standard NG interface.

* **RU (Radio Unit):** The physical radio interface (the "air") will be simulated. We will use the **ZMQ (ZeroMQ)** driver from srsRAN. This driver acts as a "virtual RU" that replaces radio frequency (RF) transmission with a local network channel (a TCP socket).

* **UE (User Equipment):** The **srsRAN UE** (in a container) will simulate the mobile device. Instead of using radio, it will connect directly to the gNB via the ZMQ channel, completing the simulated radio link.

The following diagram illustrates how these containers will interact:
![alt text](figures/diagramArchitecture.png)

Aquí tienes la guía paso a paso para desplegar el Core Network.

-----

Here is the translation of the text into English, maintaining the original formatting:

## Step 1: Deploying the Core Network (Open5GS)

In this first step, we will deploy the "brain" of our 5G network. We will use **Open5GS** because it is a complete open-source implementation of the 5G Core, designed to be easily deployed using Docker.

### 1\. Clone the Open5GS Repository

The easiest way to deploy Open5GS is by using a repository prepared for deployment. This repository requires us to build the Docker images from the source, instead of downloading them.

First, open a terminal on the host machine and clone the repository:

```bash
git clone https://github.com/wocn-unicamp/5g-network-udelar.git
```

### 2\. Start the Core Services

Navigate to the directory you just cloned and the `docker-open5gs` subfolder to start all the 5GC services (AMF, SMF, UPF, etc.).

```bash
cd 5g-network-udelar/docker-open5gs
```

Compile the base image using the `make` command. This command compiles the main Open5GS source code and installs it into a clean Docker image (based on Ubuntu). It is required for all microservices (AMF, SMF, etc.).

```bash
make base-open5gs
```

Once the base image is built, we can bring up the entire network. The docker-compose.yml file located at `compose-files/metrics/docker-compose.yaml` is a configuration that brings up not only the 5G Core, but also the WebUI, a monitoring stack (Prometheus/Grafana), and a RAN simulator (UERANSIM).

```bash
docker compose -f compose-files/metrics/docker-compose.yaml --env-file=.env up -d
```

Note:

* `docker compose` reads the `docker-compose.yml` file and brings up all the containers defined in it. The `-d` (detached) flag runs the process in the background.
* The first time you run this, Docker will download all the necessary images. It may take a few minutes.
* Tear down the basic deployment ```docker compose -f compose-files/basic/docker-compose.yaml --env-file=.env down```

### 3\. Verify the Deployment

You can verify that all the 5G Core containers are running correctly:

```bash
docker ps
```

Note: You should see a long list of services in an `Up` state, including:

* webui (The graphical interface)
* amf, smf, upf, nrf, db (The 5G Core)
* prometheus, grafana (The monitoring stack)

### 4\. Register a Subscriber (UE)

Your Core network is running, but it doesn't "know" any devices. We must manually register the phone (UE) that we will simulate later with srsRAN.

1. **Access the Web Interface:**
    Open your web browser and go to `http://localhost:9999`. This is the Open5GS administration interface (WebUI). To log in, use: User: `admin` and Password: `1423`

2. **Navigate to Subscribers:**
    In the left-hand menu, click on **"Subscribers"**. You will see a list (probably empty).

3. **Add a New Subscriber:**
    Click the green **"Add"** button.

4. **Fill in the Subscriber's Data:**
    You must fill in the fields with the exact values that our srsRAN UE will use. **It is crucial that this data matches.**

    > **Important:** Copy and paste the following exact values:
    > * **IMSI:** `999700000000001` (This is the subscriber's unique ID)
    > * **Authentication (K):** `465B5CE8B199B49FAA5F0A2EE238A6BC`
    > * **Authentication (OPc):** `E8ED289DEBA952E4EB36576D2F9C09A3`
    > In the **"Slice"** section:
    > * Click **"Add Slice"**.
    > * **SST:** `1`
    > * **SD:** `000001`
    > * **Default Selection:** `Default`
    > In the **"Access Point Names (APN)"** section:
    > * Leave the default APN `internet`.

5. **Save the Subscriber:**
    Click **"Save"**.
