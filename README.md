# ![](images/openshift.png)

<br><br><br>

![](images/hpe.png)
**Patrick D'appollonio**
<small>Hewlett-Packard Enterprise USA
HPE Datacenter Care - Center of Excelence</small>

---

<!-- page_number: true -->

# OpenShift?

OpenShift is an all-in-one solution to orchestrate workloads based on containers. It uses **Kubernetes** (from Google) internally as well as **Docker** to perform, among other features:

* Application builds
* Deployments
* Scaling
* Health management
* Orchestration
* Self-service platform

---

# Components of OpenShift

A minimal OpenShift installation is based on a couple of main applications running:

* **Docker engine:** This will manage the Docker container platform, as well as the Docker Registry features.
* **Kubernetes:** The core of the platform: this is the app that will handle and manage the container lifecycle inside OpenShift. 
* **Docker registry:** It's separated from Docker, because Docker by itself doesn't include a registry server. OpenShift needs an internal OpenShift registry server to maintain a temporary copy of the builds.

---

* **Etcd:** A key-value datastore to persist certain cluster details / state across all of the OpenShift platform. 
* **OpenShift router:** The OpenShift router is based on HAProxy, it's the application running on the master nodes which will take a request from an external account and route it through the OpenShift platform directly to the container that's supposed to serve it.
* **OpenShift STI / S2I:** This is an extra feature of OpenShift called _Source-to-Image_. What it does is, given a Git repo and an unknown source code, it'll take the code, detect the stack and build the project for distribution in an appropiate way. By default, no runtime is installed, but OpenShift allows an easy way to get Java, NodeJS and Ruby.

---

* **Deployer:** The _semi-HA_ feature of OpenShift is, if either a change in the codebase for an S2I project or the container / pod went down, the Deployer will redeploy a new container. This is the _stubborn_ piece of the software.
* **Docker SDN:** The software-defined network based on the Docker technology. Every time a container is created, Docker will create a software-defined network which may or may not use for communication purposes between containers. By linking two containers, you're specifying that they should resolve each other by container name as the host name.
* **Authentication:** The current OpenShift authentication is based on `HTPasswd` which you can use to create development accounts in the OpenShift installation.

---

* **Web Console:** The Web Console is the easy-to-use way to manage the OpenShift installation. You can do any development task from the UI, like deploying new code, manage the number of pods running (upscaling / downscaling) as well as URI Endpoints for running pods.
* **The `oc` command:** The `oc` command is a CLI application to manage both the development flow as well as the administration flow of OpenShift. The `oc` command is the most complete way of OpenShift administration and automation.
* **REST API:** To extend the power of the OpenShift platform, OpenShift also has an API you can use for things like deployments, S2I and so on.

---








































