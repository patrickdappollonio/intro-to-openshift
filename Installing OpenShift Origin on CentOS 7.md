# Installing OpenShift Origin on CentOS 7

### System setup

Check that SELinux is enabled and enforced, with targeted type. When checking the file `/etc/selinux/config` it should look like this (double check that the `SELINUX` and `SELINUXTYPE` matches the comments above):

```bash
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=enforcing
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted  
```

You'll also need to **install some extra dependencies** that the OpenShift installation will use, run the following command:

```bash
yum install wget git net-tools bind-utils iptables-services bridge-utils bash-completion kexec sos psacct
```

It's also advisable to update the system by running `yum update` and installing all the extra dependencies.

The last step is to **configure the hostname** of the machine to match the fqdn one. In RPM environments, this is done by setting `hostnamectl` to a proper value. Use the following commands in the following order (because order matters, either it'll override certain defaults):

```bash
# Check the current configuration, verify both the 
# "transient" hostname as well as the "static" hostname
hostnamectl status

# Change the transient hostname first to be the first part of
# the given FQDN
hostnamectl set-hostname "master"

# Change the static hostname to be the full FQDN
hostnamectl set-hostname "master.example.com" --static
```



### Install and Configure Docker

Get both of the binaries needed to build OpenShift Origin, `docker-engine-1.12` and `docker-engine-selinux-1.12` [from Docker YUM repo](https://yum.dockerproject.org/repo/main/centos/7/Packages/), and install both of the rpm files using:

```bash
yum install docker-engine-1.12.6-1.el7.centos.x86_64.rpm docker-engine-selinux-1.12.6-1.el7.centos.noarch.rpm
```

Alternatively, you can create a yum repository pointing to the Docker CentOS 7 repo, by issuing the following command:

```bash
cat > /etc/yum.repos.d/docker.repo << 'EOF'
[docker]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
```

Once you added the repository to the yum available repositories, you can simply execute:

```bash
yum -y install docker-engine
```

Then, once Docker is installed, we need to configure it to accept an insecure registry for OpenShift. To do that, we need to modify the file `/etc/sysconfig/docker` and add the following configuration:

```bash
OPTIONS="--selinux-enabled --insecure-registry 127.0.0.1/24"
```

You'll need to generate a proper CIDR to pass. All the nodes in the OpenShift installation must be able to see the Docker Registry, so the CIDR needs to cover all of the available IPs. To calculate a custom CIDR, [go to this website](https://grox.net/utils/whatmask/) and tweak depending on your network mask.



### Installing the required OpenShift tools

The OpenShift tools can be downloaded from [Github releases for OpenShift](https://github.com/openshift/origin/releases). They're usually tagged with a green badge mentioning the _latest_ version on the left side of each release. Pick the most up-to-date version and download the file named `openshift-origin-server-vX.X.X-sha1-linux-64bit.tar.gz`, where `vX.X.X` is the version number and `sha1` is the SHA code representing the commit used to build the given version. Download it to both Masters and Nodes, and install it in a proper location:

```bash
# Download the required binaries
cd /tmp && wget https://github.com/openshift/origin/releases/download/v1.5.1/openshift-origin-server-v1.5.1-7b451fc-linux-64bit.tar.gz

# Extract them into the same folder
tar xvf openshift-origin-server-v1.5.1-7b451fc-linux-64bit.tar.gz
cd openshift-origin-server-v1.5.1-7b451fc-linux-64bit

# Copy only the needed files to a location in the path, in this case, /usr/local/bin
mv k* o* /usr/local/bin

# Test that everything went right by calling openshift version
openshift versioncl
```



### Generate configuration files for Masters and Nodes

Now that we have the OpenShift tools installed and Docker 1.12 installed, it's time to create the Setup needed to use with OpenShift. Things like configuration files, certificates and some other extra files are the core of the setup and fortunately enough, OpenShift CLI allows you to create those in an easy way.

From now on, we will assume we're doing an OpenShift installation with 1 master and 2 nodes, the master named `master.example.com` with IP `16.0.1.2`, and the nodes called `node-1.example.com` with IP `16.0.1.3` and `node-2.example.com` with IP `16.0.1.4` respectively.

To create the configuration, execute the following steps:

```bash
# Create a folder in master and nodes to hold the configuration
mkdir -p /opt/openshift/openshift.local.config/ && cd /opt/

# [MASTER ONLY] generate the configuration for master
# The first parameter, --write-config, allows to specify where to store the master config
# The second and third parameter specify the user-facing URL and the internal IP address
# that the nodes can ping to, to connect and handle openshift workload
openshift start master \
    --write-config=/opt/openshift/openshift.local.config/master \
    --public-master='https://master.example.com:8443' \
    --master='16.0.1.2'

# [MASTER ONLY] generate the configuration for all nodes (including master!), repeat for as many
# nodes as you need to setup. The CLI app only reads from the dir "openshift.local.config"
# so we need to cd into the /opt/openshift folder
cd /opt/openshift/ 
oadm create-node-config \
    --node-dir=/opt/openshift/openshift.local.config/node-node-1.example.com \
    --node=<node-fqdn-here> \
    --hostnames=node-1.example.com,16.0.1.3 \
    --master='https://master.example.com:8443'
```

With the commands above, you'll get pretty much like the following folder structure:

```txt
[root@master]# tree openshift.local.config/ -L 1
openshift.local.config/
├── master 
├── node-master.example.com
├── node-node-1.example.com
└── node-node-2.example.com
```

To copy the configuration files from the master to the nodes, you need to have an SSH key enabled and active under `~/.ssh/id_rsa` (with his public key at `~/.ssh/id_rsa.pub`). You can copy those keys to the nodes by executing:

```bash
for host in master.example.com \
    node-1.example.com \
    node-2.example.com; \
    do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; \
    done
```

Then you'll be able to `scp` your files from the master to the nodes. Execute the following command to copy the folder to its destination:

```bash
scp -r /opt/openshift/openshift.local.config/node-node-1.example.com \
    root@node-1.example.com:/opt/openshift/openshift.local.config/node-node-1.example.com
```

This will copy the node configuration files from the folder in master to the same destination into all the nodes. Repeat for as many nodes as you have.

Now it's possible to launch the master and the nodes by issuing the following commands

```bash
# on the master (we initialize both the master and the node)
openshift start \
	--master-config=/opt/openshift/openshift.local.config/master/master-config.yaml \
	--node-config=/opt/openshift/openshift.local.config/node-master.example.com/node-config.yaml

# on each node, changing the hostname as corresponds
openshift start node \
	--config=/opt/openshift/openshift.local.config/node-node-1.example.com/node-config.yaml
```



### Automating the startup with a Startup Script + systemd

Let's create a bash script to automate the startup process. Let's create a file at `/opt/openshift/startup.sh` and add the following contents to it, depending if it's master or node. For master:

```bash
#!/bin/bash
openshift start master \
	--config=/opt/openshift/openshift.local.config/master/master-config.yaml
```

For a node:

```bash
#!/bin/bash
openshift start node \
	--config=/opt/openshift/openshift.local.config/node-node-1.example.com/node-config.yaml
```

So both master and all nodes will have a file on `/opt/openshift/startup.sh` which starts the OpenShift server. Give them all permissions by doing `chmod u+x /opt/openshift/startup.sh`.

Let's create a systemd service for it now. On all master and nodes, create a file at `/etc/systemd/system/openshift.service` with the following contents:

```ini
[Unit]
Description=OpenShift Origin Startup
After=docker.service
Requires=docker.service

[Service]
Type=simple
ExecStart=/opt/openshift/startup.sh
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

Then to activate the service on `systemd`, just run `systemctl daemon-reload` to pick up the new `.service` file, and then run `systemctl start openshift`. There's an extra step to add if you want this process to run once the server turns on, which is `systemctl enable openshift`: make sure to run it on both the server and the nodes.

After running that command, you should have the option to see your OpenShift installation by going to your browser and accessing: `https://master.example.com:8443/`. It'll require an username and password, and since we haven't generated any users yet, you can log in with any fake username and password to manage OpenShift as an user rather than an administrator.



## Further configuration of the OpenShift environment

### Logging in as `system:admin` into OpenShift

`system:admin` is the administrator account used for all administrative task of OpenShift. Since we haven't configured any authentication yet, the only way to log in is by passing to the `oc` command the proper parameters, since by default it's not possible to log in into OpenShift directly with `system:admin`: to do it you need to use certificate authorization.

So the two things we need to provide to the `oc` command so we can issue commands as `system:admin` is the Kubernetes configuration file -- so it can authenticate on our behalf against the Kubernetes cluster -- and the CA Bundle generated during the creation of the config files, which represent the `system:admin` user.

To pass those values, you need to specify two environment variables: `$KUBECONFIG` and `$CURL_CA_BUNDLE`. If you're planning on running this from a machine external to the OpenShift cluster, then it's better to copy the files those environment variables link to, to your own machine.

```bash
export KUBECONFIG=/opt/openshift/openshift.local.config/master/admin.kubeconfig
export CURL_CA_BUNDLE=/opt/openshift/openshift.local.config/master/ca.crt
```

After setting those values -- which you can also make more permanent by adding them to `.bashrc` or similar -- then you can log in to the OpenShift instance from the master -- and we will be using the master for administrative tasks, although you can get the OpenShift CLI tools and use them from anywhere you want by just setting those environment variables. 

A better approach is to make those environment variables more permanent by setting them in `/etc/profile.d`:

```bash
cat > /etc/profile.d/openshift.sh << 'EOF'
export OPENSHIFT_CONF=/opt/openshift/openshift.local.config
export KUBECONFIG=$OPENSHIFT_CONF/master/admin.kubeconfig
export CURL_CA_BUNDLE=$OPENSHIFT_CONF/master/ca.crt
EOF
chmod 755 /etc/profile.d/openshift.sh
source /etc/profile.d/openshift.sh
```

By running those commands, you'll create both of the required environment variables as well as set a handy environment variable to point to the `$OPENSHIFT_CONF` directory. It'll also source the directory for you, so those variables will be already in your session.

To log in from the master itself, do:

```bash
$ oc login -u system:admin
Logged into "https://16.0.1.2:8443" as "system:admin" using existing credentials.

You have access to the following projects and can switch between them with 'oc project <projectname>':

  * default
    kube-system
    openshift
    openshift-infra
    test
    
Using project "default".
```

You'll confirm it works because you get the confirmation message after running the command. You'll also see:

* The IP address of the master node, under HTTPS, under port 8443 as expected

* A confirmation on what user was used to log in, in this case, `system:admin`

* A list of projects -- also called namespaces -- where projects can live

* The current namespace or project enabled for the given session




### Configuring deployments on master and nodes using Labels

One of the next things we will do will be to deploy the OpenShift internal Docker Registry as well as the HAProxy router. But in order to do so those two services are required to be installed in the Master node -- which is also an OpenShift node. Technically, though, it's not recommended to publish services on master beside these two, so Master can only focus in one thing: serve OpenShift.

So in order to do this, what we need to do is to label our nodes. Labels serve two purposes: in one side, it's an organization feature to allow you to easily distinguish details about a node; in the other side, it allows us to deploy stuff to nodes using an specific label. If you think about this in a different way, assume you have two different types of nodes: RAM-centric and CPU-centric. You could create labels called `power=ram` or `power=cpu`, and then later on you can deploy RAM-intensive applications by saying only deploy it in those labeled as `power=ram`.

By default, we will label them by `region`, even when we don't have more than a single region, but it'll allow us to also say that the Master is off-limits of scheduling by calling its region `infra`.

The first thing we will do it's to update the OpenShift master configuration, so open the master config YAML file located at `/opt/openshift/openshift.local.config/master/master-config.yaml` and find for `projectConfig`. Change the `defaultNodeSelector` field to match what you see below:

```yaml
projectConfig:
    defaultNodeSelector: "region=primary"
```

This will tell that, by default, we want OpenShift to publish our projects in those nodes whose region is `primary` (our two nodes). 

Now we can proceed to label our nodes. In order to do that, go to the Master terminal, and using the `oc` command, first, list all the nodes:

```bash
$ oc get nodes
NAME                 STATUS    AGE
master.example.com   Ready     21m
node-1.example.com   Ready     21h
node-2.example.com   Ready     21h
```

Then, knowing those nodes, we will tag them accordingly:

```bash
$ oc label node master.example.com region=infra
node "master.example.com" labeled

$ oc label node node-1.example.com region=primary
node "node-1.example.com" labeled

$ oc label node node-2.example.com region=primary
node "node-2.example.com" labeled
```

We can see if this works by limiting the `oc get nodes` command by passing the label selector:

```bash
$ oc get nodes -l region=infra
NAME                 STATUS    AGE
master.example.com   Ready     21m

$ oc get nodes -l region=primary
NAME                 STATUS    AGE
node-1.example.com   Ready     21h
node-2.example.com   Ready     21h
```

You can also see a more thoughtful explanation of the node by doing:

```bash
$ oc describe node master.example.com
Name:                   master.example.com
Role:
Labels:                 beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/os=linux
                        kubernetes.io/hostname=master.example.com
                        region=infra
Taints:                 <none>
CreationTimestamp:      Fri, 23 Jun 2017 12:34:56 +0000
```

You'll see here that in the `Labels:` section there's our new region label `region=infra`. The same is true for all the other nodes.

This is all on this step. In the next part, we will deploy the router and the registry, and when we do so, we will specify we want them to be deployed in the master.



### Adding a router for external access and a Docker registry

The router and the registry are two of the most important things to do when working with OpenShift. They control both access to resources deployed to OpenShift as well as allow you to create apps using the Source-to-Image method. The registry requires to store files on the master hard drive to keep important information even after a system reboot. 

Before we can go through though, we need a couple of things from the Docker registry: we need all the docker container needed to build the registry, the pod management, the S2I builder, the deployer and so on. But in order to download the appropiate one, we need to know which version of OpenShift we're running. To check it out, we can simply execute `oc version` from the terminal, which will output something like this:

```text
oc v1.5.1+7b451fc
kubernetes v1.5.2+43a9be4
features: Basic-Auth GSSAPI Kerberos SPNEGO

Server https://16.0.1.2:8443
openshift v1.5.1+7b451fc
kubernetes v1.5.2+43a9be4  
```

The only part we're interested in is in the section below under `Server`. Grab the version from there by taking just the `v` followed by the 3 digits separated by a dot, until the `+` sign comes up. Our version then is `v1.5.1`. Then, let's create an `$OPENSHIFT_VERSION` environment variable:

```bash
export OPENSHIFT_VERSION="v1.5.1"
```

Or modify your environment configuration (depending on if you did this step):

```bash
echo 'export OPENSHIFT_VERSION="v1.5.1"' >> /etc/profile.d/openshift.sh
source /etc/profile.d/openshift.sh
```

Then, the next step is to download the Docker images for each one of the needed resources from the Docker Hub, and set them to the specific version we just got:

```bash
docker pull openshift/origin-pod:$OPENSHIFT_VERSION
docker pull openshift/origin-sti-builder:$OPENSHIFT_VERSION
docker pull openshift/origin-docker-builder:$OPENSHIFT_VERSION
docker pull openshift/origin-deployer:$OPENSHIFT_VERSION
docker pull openshift/origin-docker-registry:$OPENSHIFT_VERSION
docker pull openshift/origin-haproxy-router:$OPENSHIFT_VERSION
```

This will download all the required OpenShift containers needed to run deployments to OpenShift. Now let's go back to the storage. Create a folder inside `/opt/openshift` to store our registry:

```bash
# Create the directory
mkdir -p /opt/openshift/registry

# Change the SELinux security context for this folder
chcon -Rt svirt_sandbox_file_t /opt/openshift/registry

# Create an OpenShift administrative policy for the registry
oadm policy add-scc-to-user privileged -z registry
```

Then the last step is to actually deploy the registry by issuing the following command. Note the selector parameter to define we want the router to run in master:

```bash
oadm registry \
	--service-account=registry \
	--mount-host=/opt/openshift/registry \
	--selector='region=infra'
```

Which will output something like this:

```bash
--> Creating registry registry ...
    serviceaccount "registry" created
    clusterrolebinding "registry-registry-role" created
    deploymentconfig "docker-registry" created
    service "docker-registry" created
--> Success   
```

That process will generate a deploy flow and an actual registry. Both of them are pods in the OpenShift / Kubernetes environment, which you can check by issuing:

```bash
$ oc get pods
NAME                       READY     STATUS              RESTARTS   AGE
docker-registry-1-82tl1    0/1       ContainerCreating   0          5s
docker-registry-1-deploy   1/1       Running             0          58s
```

This will be a likely output when no time has passed since the `oadm registry` command was issued. Here it shows a pod called `docker-registry-1-deploy` which is the deployment flow for the actual registry. You'll see that the status is `Running` which means it's an ongoing operation. All of the deployment flows will generate a pod with the suffix `-deploy`. The pod above the `deploy` one is the actual registry. In this case it's status is `ContainerCreating` which means the container is being created with all the instructions from the `deploy` pod.

After a couple of seconds once the whole deploy flows ran, the output of the command will change:

```bash
$ oc get pods
NAME                      READY     STATUS    RESTARTS   AGE
docker-registry-1-82tl1   1/1       Running   0          3m
```

Which will confirm the registry is running. You can also execute the following command to see the service running -- which matches the service name generated in the `oadm registry` step above:

```bash
$ oc get svc docker-registry
NAME              CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
docker-registry   172.30.63.218   <none>        5000/TCP   5m
```

Since now we have a valid Docker Registry, working internally, we can pull down the HAProxy router for OpenShift. Execute the following commands:

```bash
# Create the OpenShift administrative policy for the router
oadm policy add-scc-to-user hostnetwork -z router

# Then create the actual router process by deploying it to OpenShift
# Since we only have one master, `replicas` will be 1
oadm router router --replicas=1 --service-account=router --selector='region=infra'
```

The output of the last command will be something familiar, like follows:

```bash
info: password for stats user admin has been set to abc123
--> Creating router router ...
    serviceaccount "router" created
    clusterrolebinding "router-router-role" created
    deploymentconfig "router" created
    service "router" created
--> Success
```

Do note that this output also contains a password. HAProxy includes a stats page that can be made visible and you can use to see some health stats related to the OpenShift routes. In this case, the password was set to `abc123`.

If you were quick enough and ran `oc get pods`, you should be able to see pretty much what we saw before: a deploy pod and the actual router being deployed. Once the router has been successfully deployed you should see two pods that'll stick all the way through in our OpenShift installation:

```bash
$ oc get pods
NAME                      READY     STATUS    RESTARTS   AGE
docker-registry-1-82tl1   1/1       Running   0          10m
router-1-0qh8c            1/1       Running   0          1m
```

And same as before, you can also see the service running by asking for it:

```bash
$ oc get svc router
NAME      CLUSTER-IP      EXTERNAL-IP   PORT(S)                   AGE
router    172.30.38.199   <none>        80/TCP,443/TCP,1936/TCP   22m 
```

Now there's also one extra command related to all the "default" projects running in our OpenShift environment. We can see the router, the registry, and of course, Kubernetes, by running the `oc status` command. The output looks like this:

```bash
$ oc status
In project default on server https://16.0.1.2:8443

svc/docker-registry - 172.30.63.218:5000
  dc/docker-registry deploys docker.io/openshift/origin-docker-registry:v1.5.1
    deployment #1 deployed 13 minutes ago - 1 pod

svc/kubernetes - 172.30.0.1 ports 443, 53->8053, 53->8053

svc/router - 172.30.38.199 ports 80, 443, 1936
  dc/router deploys docker.io/openshift/origin-haproxy-router:v1.5.1
    deployment #1 deployed 3 minutes ago - 1 pod

View details with 'oc describe <resource>/<name>' or list everything with 'oc get all'.
```

You'll see here a nice picture of the whole environment: the Docker registry is running and it has an internally networked IP address, you can also see it was downloaded from `docker.io/openshift/origin-docker-registry` and how long ago it was deployed -- in this case, 13 minutes ago. The same is true for the router. In the case of Kubernetes, that's a different story, since Kubernetes gets deployed along with OpenShift when we installed it.



### Configuring the wildcard DNS record to resolve to the apps

A requirement of OpenShift to serve web applications using the HAProxy router is to have a wildcard DNS record which serves as domain names for the routing of the different apps deployed to OpenShift. In practice, this is an `A` record that looks like this:

```text
*.apps.master.example.com   IN A   16.0.1.2
```

Where `16.0.1.2` is the IP address of the OpenShift master where the router resides. Then later on when you deploy applications, those will be live based on the application name set in OpenShift, so for a project called `example`, then the URL to access it will be `example.apps.master.example.com`. This domain can be as well as can not be the same name as the `master.example.com` host. It all depends on your DNS service provider.

Now, to tell OpenShift how to implement this domain name, we will need to change the configuration, so open the configuration file at `/opt/openshift/openshift.local.config/master/master-config.yaml`, then look for the YAML configuration called `routingConfig`, which will look like this:

```yaml
routingConfig:
  subdomain: router.default.svc.cluster.local
```

Change the `subdomain` from `router.default.svc.cluster.local` to `apps.master.example.com`, save the file and restart the service by issuing `



### Adding an extra layer of security with HTPasswd

Now, by default, OpenShift itself runs with an identity provider `AllowAll` which means any non-empty username and password will allow you to log in. We want to change that so only the people who have an active username and password can log in. Even when there are multiple options for log in flows, we will use `HTPasswd` since it's the easiest one to manage and control.

In order to use `HTPasswd`, we will need the package `httpd-tools` which is able to generate `HTPasswd` files. Do it by executing:

```bash
yum install httpd-tools
```

As an extra comment, only MD5, bcrypt, and SHA encryption types are supported to use with OpenShift. MD5 encryption is recommended, and is the default for `htpasswd`. Plaintext and crypt hashes are not currently supported.

Now, assuming we will have a list of users we want them to log in, we will create an account for each one of them using the `htpasswd` command. Execute the following:

```bash
# create a directory to store the passwords
# (you can also use an existent directory, even better if it's encrypted)
mkdir -p /opt/openshift/accounts/

# then create a new username with password on the file below
# where "developer" is the username (htpasswd will prompt for the
# password), and /opt/openshift/accounts/users.htpasswd is the 
# location of the file where we will store the accounts
htpasswd -c /opt/openshift/accounts/users.htpasswd developer

# to remove an account, like the "developer" account before, you
# can issue the following command
htpasswd -D /opt/openshift/accounts/users.htpasswd developer
# you can also manually edit the file and remove the line related
# to the "developer" account

# to update the password of an existent user, you can do
htpasswd /opt/openshift/accounts/users.htpasswd developer
```

If we inspect the file once we create the user `developer`, and assuming a password of `123`, then the file will look like this:

```htpasswd
developer:$apr1$AKqbsqKg$rz8v0q2H1RbuIplTVEJlq0
```

Now, to make that username and password available to use on OpenShift we need to change the configuration and set the authentication to use `HTPasswdPasswordIdentityProvider`. Let's edit the master configuration file located at `/opt/openshift/openshift.local.config/master/master-config.yaml`, find the section called `oauthConfig`, which will look like this:

```yaml
oauthConfig:
  ...
  identityProviders:
  - challenge: true
    login: true
    mappingMethod: claim
    name: anypassword
    provider:
      apiVersion: v1
      kind: AllowAllPasswordIdentityProvider
```

And change it to this:

```yaml
oauthConfig:
  ...
  identityProviders:
  - challenge: true
    login: true
    mappingMethod: claim
    name: demo_htpasswd_provider 
    provider:
      apiVersion: v1
      kind: HTPasswdPasswordIdentityProvider
      file: /opt/openshift/accounts/users.htpasswd 
```

The next step is to restart OpenShift in the master to make the changes available again. To do so, just issue a `systemctl` command: `systemctl restart openshift`. If we now try to log in with any username or password via the UI at `https://master.example.com:8443` it will warn us with a message:

> Invalid login or password. Please try again.

But if we log in with the username we created before, it'll go straight to the control panel where we can see all our options.

We can definitely change the password of the user if we see fit using the command mentioned above, and we can also prevent anyone from logging in by just emptying the file, since each line belongs to one user.



### Improving Openshift: Adding default templates

An extra feature to start easily new projects is the fact that we can have multiple templates laying around, ready to be "filled" with actual data and start services that way. By default, OpenShift doesn't come with any template available, but installing them it's pretty straightforward since they're just JSON files. Execute the following steps to import them:

```bash
# Let's go to a temporary directory
$ cd /tmp

# Download Openshift-Ansible (the OpenShift Ansible installer)
$ git clone https://github.com/openshift/openshift-ansible.git
Cloning into 'openshift-ansible'...
remote: Counting objects: 56132, done.
remote: Compressing objects: 100% (30/30), done.
remote: Total 56132 (delta 5), reused 32 (delta 2), pack-reused 56095
Receiving objects: 100% (56132/56132), 14.64 MiB | 2.70 MiB/s, done.
Resolving deltas: 100% (34243/34243), done.     

# Then let's import each one of the templates available there
$ cd openshift-ansible/roles/openshift_examples/files/examples/latest/ && \
    for f in image-streams/image-streams-centos7.json; \
        do cat $f | oc create -n openshift -f -; done && \
    for f in db-templates/*.json; \
        do cat $f | oc create -n openshift -f -; done && \
    for f in quickstart-templates/*.json; \
        do cat $f | oc create -n openshift -f -; done && \
    for f in xpaas-templates/eap*.json; \
        do cat $f | oc create -n openshift -f -; done
```

This will iterate over all the required files needed and add them as templates to the OpenShift installation for later use. This will bring things like CakePHP and MySQL to be available to be installed with ease, with just a couple of clicks.