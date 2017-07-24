# Installing OpenShift using OpenShift-Ansible

### System setup

#### SELinux

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

#### Extra dependencies + EPEL Repository for Yum

You'll also need to **install some extra dependencies** that the OpenShift installation will use, run the following command:

```bash
yum install wget git net-tools bind-utils iptables-services bridge-utils bash-completion kexec sos psacct
```

It's also advisable to update the system by running `yum update` and installing all the extra dependencies.

Additionally, you definitely should install the EPEL repo for Yum. Let's check first if it's there. `cd` into `/etc/yum.repos.d` and find there if there's a file called `epel.repo`. If so, let's make sure it's the repo we want so delete it by running `rm -rf /etc/yum.repos.d/epel.repo` and then add the EPEL repo by running:

```bash
yum -y install \
    https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
```

This will make the EPEL repo available for our system. Once we have that, we need to install Ansible to use the Ansible installation method, as well as the libraries for Python OpenSSL 3:

```bash
yum -y install ansible pyOpenSSL
```

Finally, a last step, is to confirm the CentOS Extra repo is enabled. To do so, check the contents of the file `/etc/yum.repos.d/CentOS-extras.repo` file -- it may have a different name, but it's always inside `/etc/yum.repos.d`. Check for a line `enabled=1`. If it's set to zero, then it means it's disabled. Please enable it by setting it to 1, run `yum update` again and you should be good to go!



#### Docker for container management

Now, once we have the previous packages in place, it's time to install Docker. To do so, please just run:

```bash
yum install docker
```

Do please check that the version you're installing comes from the CentOS `extras` repo. Some extra packages may likely be installed like `container-selinux`.

One last step: you need to add the future OpenShift internal Docker registry as insecure into the docker configuration. To do so, edit `/etc/sysconfig/docker` and add to that file -- or create it if it doesn't exists:

```ini
OPTIONS='--selinux-enabled --insecure-registry 172.30.0.0/16'
```

Then save and exit. There may be a couple of extra settings you need to do before the installation, such as configuring a Proxy and if so, also set the insecure internal registry to not use a proxy. Once that's ready, enable and start Docker:

```bash
$ systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service. 

$ systemctl start docker
```

#### NetworkManager

The last requirement for our environment is to have `NetworkManager` enabled and started in our environment. To install and enable it, execute the following commands:

```bash
yum install -y NetworkManager
systemctl enable NetworkManager
systemctl start NetworkManager
```

This should download, enable and start the `NetworkManager` service, required by the Ansible installation method.



#### Cloning the OpenShift Ansible repository

Once everything has been done before, we're ready to clone the OpenShift Ansible installer from Github. To do so, go to your home directory and then perform the `git clone` process:

```bash
cd ~/ && git clone https://github.com/openshift/openshift-ansible
```



### Configuring the Ansible OpenShift installation

From now on, we will assume we're working on a single master, two node OpenShift installation. We will call the master `master.example.com` with IP `192.168.0.100`, then the first node will be `node-1.example.com` with IP `192.168.0.101` and the second node will be `node-2.example.com` with IP `192.168.0.102`. Once OpenShift is running, all public-facing apps will be available under the domain `osapps.example.com`, so for an app called `demo`, the hostname `demo.osapps.example.com` will be available.

Now what you need to do is update the OpenShift inventory for Ansible. The Ansible inventory file defaults to `/etc/ansible/hosts`. Let's modify that file to make it look like this:

```bash
# Create an OSEv3 group that contains the masters and nodes groups
[OSEv3:children]
masters 
nodes

# Set variables common for all OSEv3 hosts
[OSEv3:vars] 
ansible_ssh_user=myuser
ansible_become=true 
openshift_deployment_type=origin
openshift_master_default_subdomain=osapps.example.com
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/master/htpasswd'}]

# host group for masters
[masters]   
master.example.com

# host group for etcd 
[etcd]        
master.example.com

# host group for nodes, includes region info
[nodes]
master.example.com openshift_node_labels="{'region': 'infra'}" openshift_schedulable=true
node-1.example.com openshift_node_labels="{'region': 'primary'}"
node-2.example.com openshift_node_labels="{'region': 'primary'}" 
```

There are some extra settings [you can see in the documentation](https://docs.openshift.org/latest/install_config/install/advanced_install.html#configuring-ansible), but a pretty standard setup should be like the one mentioned above. 

Do note that you need to change hostnames as well as the Username you'll use to SSH into the different nodes and the master. If you prefer logging in as `root` then you can change `ansible_ssh_user` to `root` and comment out the line `ansible_become`. To log in via SSH, you should definitely enable passwordless SSH. There are some options to add a password here, but it's out of the scope of this.

Additionally, we've marked the `openshift_schedulable` value to `true` in the master. This is because since we've added regions to our setup, then only those nodes in the `infra` region will be able to contain an OpenShift router or registry. If you failed to do this part, but still set those regions, you can toggle the schedulability of the master by issuing:

```bash
oadm manage-node <node-name> --schedulable=true
```

Where `<node-name>` is the name of the node given by the `oc get nodes` command.

Also remember we're setting the `openshift_master_identity_providers` which setups authentication for our OpenShift instance from day one. Per YAML requirements, this line has to be a single line, which means you cannot use linebreaks to pretty-print it. Once you saved the previous file, we will come back later to setup authentication.

Now it's time to run the Ansible playbook, but first, let's check everything is in order to proceed. To check if Ansible is able to access all the hosts, run the following command:

```bash
ansible -m ping all
```

This will instruct Ansible to take the `/etc/ansible/hosts` file, parse it -- failing if the syntax is not YAML or if we made a mistake -- and then going to each one of our servers and check if there's connection against it. 

This will also connect via SSH to each one of the servers, so we will need a private and public key in the master where we're running this script. Check if you have one by issuing `cat ~/.ssh/id_rsa`. If you see the message with `BEGIN RSA PRIVATE KEY` and a long wall of text, you're good to go. If not, create an SSH key **with no password** and save it by doing:

```bash
# Follow the instructions of this command below to create
# an SSH key. Remember *not to use password* so the process
# is fully automated.
$ ssh-keygen 

# Once the SSH key generation is done, check that you have a key with the
# *BEGIN RSA PRIVATE KEY* message available by showing it on the screen
$ cat ~/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAKCAQEA1ZjZhNS1GbBTSFS68Hu9Gkht/fxMIA6kh8OP6zXcRjV9zPL6
                 ... trimmed for brevity ...
w4XFqzCQkPlEN9rJSwsrcTtXusOFO0iw8Ta8QDFsXjP8Q9MxSHM6bZI=
-----END RSA PRIVATE KEY-----

# Once the private key is in place, let's copy the public key
# from the master to all nodes -- master included -- so Ansible
# can connect to them. Replace each node hostname down below:
$ for host in master.example.com \
    node-1.example.com \
    node-2.example.com; \
    do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; \
    done
```

 Now let's try to run the `ansible ping` command again. You should see a similar output like the one below:

```bash
$ ansible -m ping all
node-1.example.com | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
master.example.com | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
node-2.example.com | SUCCESS => {
    "changed": false,
    "ping": "pong"
}    
```

The order of the hostnames really doesn't matter. As long as all of them return green and they look like the output above, then we're all set. We can run now the Ansible playbook with our hostnames -- that we know we worked because we just issued a `ping` command to all of them -- and proceed with the installation. Let's keep the party going by now running the advanced installation:

```bash
ansible-playbook ~/openshift-ansible/playbooks/byo/config.yml
```

You will see a long run of commands going on the screen. This is Ansible working. What it'll do it'll take the configuration as declared by the Ansible playbook and apply it to both the master and the nodes. It'll also create the configuration needed as well as any other extra configuration. You'll see command outputting `changed` or some of them printing `skipped`. This is normal on an Ansible run, since Ansible first will check if everything needed is in place, and if, say, something is already installed, it'll skip the installation.

If the operation fails, a summary will be presented on screen. There are some pre-checks that may cancel the OpenShift installation, part of the requirements of OpenShift. You can bypass some of them -- as we already did with the Docker version format -- but it's advised not to.

Once the installation is ready you should see a `PLAY RECAP` message on screen, telling you how many things the advanced installation changed.  To verify it worked, check if all the nodes are available:

```bash
$ oc get nodes
NAME          STATUS                     AGE
172.16.1.32   Ready                      2h
172.16.1.33   Ready,SchedulingDisabled   2h
172.16.1.34   Ready                      2h
```

You'll note that we have our 3 nodes with random IP addresses as names. One of them has a status of `SchedulingDisabled`. That tells us that's the master node, since it'll only run the OpenShift environment and nothing else.



#### Configuring authentication

One of the things we did in the configuration of the Ansible hosts file was to setup an `HTPasswd` authentication mechanism. When no option is passed, OpenShift is installed with the `AllowAll` mechanism which accepts any account at all to log in through the UI. There are multiple options to setup Authentication like LDAP or similar, but the most simple one is actually with `HTPasswd` from Apache.

In order to use `HTPasswd`, we will need the package `httpd-tools` which is able to generate `HTPasswd` files. Do it by executing:

```bash
yum install httpd-tools
```

As an extra comment, only MD5, bcrypt, and SHA encryption types are supported to use with OpenShift. MD5 encryption is recommended, and is the default for `htpasswd`. Plaintext and crypt hashes are not currently supported.

Now, assuming we will have a list of users we want them to log in, we will create an account for each one of them using the `htpasswd` command. Execute the following:

```bash
# create a new username with password on the file below
# where "developer" is the username (htpasswd will prompt for the
# password), and /opt/openshift/accounts/users.htpasswd is the 
# location of the file where we will store the accounts
htpasswd -c /etc/origin/master/htpasswd developer

# to remove an account, like the "developer" account before, you
# can issue the following command
htpasswd -D /etc/origin/master/htpasswd developer
# you can also manually edit the file and remove the line related
# to the "developer" account

# to update the password of an existent user, you can do
htpasswd /etc/origin/master/htpasswd developer
```

Make sure to make the file secure enough by giving it a `chmod` of `600` issuing `chmod 600 /etc/origin/master/htpasswd`. If we inspect the file once we create the user `developer`, and assuming a password of `123`, then the file will look like this:

```htpasswd
developer:$apr1$AKqbsqKg$rz8v0q2H1RbuIplTVEJlq0
```

Since the process of setting up authentication was part of the OpenShift installation, if we inspect the master configuration file at `/etc/origin/master/master-config.yaml` we will see that the `oauthConfig` section looks like this:

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
      file: /etc/origin/master/htpasswd
```

The next step is to restart OpenShift in the master to make the changes available again. To do so, just issue a `systemctl` command: `systemctl restart origin-master`. If we now try to log in with any username or password via the UI at `https://master.example.com:8443` it will warn us with a message:

> Invalid login or password. Please try again.

But if we log in with the username we created before, it'll go straight to the control panel where we can see all our options.

We can definitely change the password of the user if we see fit using the command mentioned above, and we can also prevent anyone from logging in by just emptying the file, since each line belongs to one user.
