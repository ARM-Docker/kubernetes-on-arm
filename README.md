## Welcome to the Kubernetes on ARM project!

#### Kubernetes on a Raspberry Pi? Is that possible?

#### Yes, now it is.    
Imagine... Your own testbed for Kubernetes with cheap Raspberry Pis and friends. 

![Image of Kubernetes and Raspberry Pi](docs/raspberrypi-joins-kubernetes.png)

#### **Are you convinced too, like me, that cheap ARM boards and Kubernetes is a match made in heaven?**    
**Then, lets go!**

## Download and build a SD Card

The first thing you will do, is to create a SD Card for your Pi. Alternatively, you may use the `.deb` deployment

Supported OSes/boards:
- Arch Linux ARM **(archlinux)**
  - Raspberry Pi 1 A, A+, B, B+, (ZERO,) armv6 **(rpi)**
  - Raspberry Pi 2 Model B, armv7 **(rpi-2)**
  - Parallella armv7, [read more](docs/parallella-status.md) **(parallella)**
  - Cubietruck, armv7 **(cubietruck)**
  - Banana Pro, armv7 **(bananapro)**
- Hypriot OS **(hypriotos)**
  - Raspberry Pi 1 A, A+, B, B+, armv6 **(rpi)**
  - Raspberry Pi 2 Model B, armv7 **(rpi-2)**

[Windows downloads](https://github.com/luxas/kubernetes-on-arm/releases/tag/v0.6.0)    
Prebuilt SD Card images for Windows:
 - archlinux/rpi/v0.6.0
 - archlinux/rpi-2/v0.6.0

```bash
# Go to our home folder, if you want
cd ~

# Install git if needed and download this project
# sudo apt-get install git
git clone https://github.com/luxas/kubernetes-on-arm

# Change to that directory
cd kubernetes-on-arm

# See which letter your newly inserted SD Card has:
sudo fdisk -l

# Get some help text about supported options
sdcard/write.sh

# Template:
sudo sdcard/write.sh /dev/sdX [board] [os] [rootfs]

# Example: Write the SD Card for Raspberry Pi 2, Arch Linux ARM and include this project's Kubernetes scripts
sudo sdcard/write.sh /dev/sdX rpi-2 archlinux kube-systemd

# The installer will ask you if you want to erase all data on your card
# Answer y/n on that question
# Prepend the command with QUIET=1 if no security check should be made
# Requires an internet connection
# This script runs in 3-4 mins
```

## `.deb` deployment
If you have already made a SD Card and your device is up and running, what can you do instead?
For that, I've made a `.deb` package, so you could install it easily

```bash
# The OS have to be systemd based, e. g. HypriotOS, Debian Jessie, Arch Linux ARM, Ubuntu 15.04

# Download the latest package
curl -sSL https://github.com/luxas/kubernetes-on-arm/releases/downloads/v0.6.2/kube-systemd.deb > kube-systemd.deb

# Requires dpkg, which is preinstalled in at least all Debian/Ubuntu OSes
dpkg -i kube-systemd.deb

# Setup the enviroinment
# It will ask which board it's running on and which OS
# It will download prebuilt binaries
# And make a swap file if plan to compile things
# If docker was installed before this, you don't have to reboot
kube-config install

# Start the master or worker
kube-config enable-master
kube-config enable-worker [master ip]

# Get some info about the node
kube-config info
```

## Setup your board from an SD Card

Boot your board and log into it.    
The user/password is: **root/root** or **alarm/alarm**      
Yes, I know. Root enabled via ssh isn´t that good.
But the task to enhance ssh security is left as an exercise to the user.      
These scripts requires root. So if you login via **alarm**, then `su root` when you´re going to do some serious hacking :)

```bash
# This script will install and setup docker etc.
kube-config install

# First, it will update the system and install docker
# Then it will download prebuilt Kubernetes binaries
# If you build kubernetesonarm/build, then all binaries will be replaced with that version

# The script will ask you for timezone. Defaults to Europe/Helsinki
# Run "timedatectl list-timezones" before to check for values

# It will ask you if it should create a 1 GB swapfile.
# If you are gonna build Kubernetes on your own machine, you have to create this

# It will ask for which hostname you want. Defaults to kubepi.

# Last question is whether you want to reboot
# You must do this, otherwise docker will behave very strange and fail

# If you want to run this script non-interactively, do this:
# TIMEZONE=Europe/Helsinki SWAP=1 NEW_HOSTNAME=mynewpi REBOOT=0 kube-config install
# This script runs in 2-3 mins
```

## (Optional) Build the Docker images for ARM

With these scripts, all required binaries are compiled.     
Proceed to [Setup Kubernetes](#setup-kubernetes) if you want to get your cluster up-and-running fast

```bash

# Build all master images
kube-config build-images

# Build all addons
kube-config build-addons

# These scripts will run approximately 45 min on a Raspberry Pi 2
# Grab yourself a coffee during the time!
```

The script will produce these Docker images:    
 - luxas/raspbian: Is a stripped `resin/rpi-raspbian` image.
 - luxas/alpine: Is a Alpine Linux image. Only 8 MB. Based on `mini-containers/base` source.
 - luxas/go: Is a Golang image, which is used for building repositories on ARM.
 - kubernetesonarm/build: This image downloads all source code and builds it for ARM.

These core images are used in the cluster:
 - kubernetesonarm/etcd: `etcd` is the data store for Kubernetes. Used only on master. [Docs](images/kubernetesonarm/etcd/README.md)
 - kubernetesonarm/flannel: `flannel` creates the Kubernetes overlay network. [Docs](images/kubernetesonarm/flannel/README.md)
 - kubernetesonarm/hyperkube: This is the core Kubernetes image. This one powers your Kubernetes cluster. [Docs](images/kubernetesonarm/hyperkube/README.md)
 - kubernetesonarm/pause: `pause` is a image Kubernetes uses internally. [Docs](images/kubernetesonarm/pause/README.md)

## Setup Kubernetes

These script are important in the setup process. 
They spin up all required services in the right order.   
If you skipped the build process, this may take ~10min, depending on your internet connection.

```bash
# To enable the master service, run
kube-config enable-master

# To enable the worker service, run
kube-config enable-worker [master-ip]
```

## Use Kubernetes (the fun part begins here)

```bash
# See which commands kubectl and kube-config has
kubectl
kube-config

# Get info about your machine and Kubernetes version
kube-config info

# Make an replication controller with an image
# Hopefully you will have some minions, so you is able to see how they spread across hosts
# The nginx-test image will be downloaded from Docker Hub and is a nginx server which only is serving the message: "<p>WELCOME TO NGINX</p>"
kubectl run my-nginx --image=luxas/nginx-test --replicas=3

# The pull might take some minutes
# See that the nginx container is running
docker ps

# See which pods are running
kubectl get pods

# See which nodes we have
kubectl get nodes

# Expose the replication controller "my-nginx" as a service
kubectl expose rc/my-nginx --port=80

# See which ip we may ping, by getting services
kubectl get svc

# See if the nginx container is working
# Replace $SERVICE_IP with the ip "kubectl get svc" returned 
curl $SERVICE_IP
# --> <p>WELCOME TO NGINX</p>

# Start dns, this will spin up 4 containers and expose them as a DNS service at ip 10.0.0.10
# 10.0.0.10 is already enabled as a DNS server in your system, see the file /etc/systemd/resolved.conf.d/dns.conf
# That file makes /etc/resolv.conf use kube-dns also outside of your containers
kube-config enable-addon dns

# See which internal cluster services that are running
kubectl --namespace=kube-system get pods,rc,svc

# Test dns
curl my-nginx.default.svc.cluster.local
# --> <p>WELCOME TO NGINX</p>

# By default, "search [domains]" is added to /etc/resolv.conf
# In this case, these domains are searched: "default.svc.cluster.local svc.cluster.local cluster.local"
# That means, that you may only write "my-nginx", and it will search in those domains
curl my-nginx
# --> <p>WELCOME TO NGINX</p>

# Start the registry
kube-config enable-addon registry

# Wait a minute for it to start
kubectl --namespace=kube-system get pods

# Tag an image
docker tag my-name/my-image registry.kube-system:5000/my-name/my-image

# And push it to the registry
docker push registry.kube-system:5000/my-name/my-image

# On another node, pull it
docker pull registry.kube-system:5000/my-name/my-image

# The registry address may be written longer if search isn't specified.
# registry.kube-system.svc.cluster.local -> registry.kube-system

# The master also proxies the services so that they are accessible from outside
# The -L flag is there because curl has to follow redirects
# You may also type this URL in a web browser
curl -L http://[master-ip]:8080/api/v1/proxy/namespaces/default/services/my-nginx

# Generic apiserver proxy URL
# curl -L http://[master-ip]:8080/api/v1/proxy/namespaces/[namespace]/services/[service-name]

# Check for open ports
netstat -nlp

# See cluster info
kubectl cluster-info

# See master health in a web browser
# cAdvisor in kubelet provides a web site that outputs all kind of stats in real time
# http://$MASTER_IP:4194

# Disable this node. This always reverts the "kube-config enable-*" commands
kube-config disable-node

# Remove the data for the cluster
kube-config delete-data
```

## Custom alternatives

If you already have set up a lot of devices and already are familiar with one OS, just grab the binaries [here](https://github.com/luxas/kubernetes-on-arm/releases/tag/v0.6.0) or pull the images from Docker Hub.

```
# Get the binaries and put them in /usr/bin
curl -sSL https://github.com/luxas/kubernetes-on-arm/releases/download/v0.6.0/binaries.tar.gz | tar -xz -C /usr/bin

# Pull the images for master
docker pull kubernetesonarm/hyperkube
docker pull kubernetesonarm/etcd
docker pull kubernetesonarm/flannel
docker pull kubernetesonarm/pause


# Pull the images for worker
docker pull kubernetesonarm/hyperkube
docker pull kubernetesonarm/flannel
docker pull kubernetesonarm/pause
```
Then check the service files here for the right commands to use: https://github.com/luxas/kubernetes-on-arm/tree/v0.6.0/sdcard/rootfs/kube-archlinux/usr/lib/systemd/system

#### However, only use this method if you know what you are doing and want to customize just for your need
#### Otherwise, use the SD Card method or deb package for an easy installation

## Addons

Two addons is available right now
 - Kubernetes DNS:
   - Every service gets the hostname: `{{my-svc}}.{{my-namespace}}.svc.cluster.local`
   - The default namespace is `default` (surprise), so unless you manually edit something it will land there
   - Kubernetes internal addon services runs in the namespace `kube-system`
   - Example: `my-awesome-webserver.default.svc.cluster.local` or just `my-awesome-webserver` may resolve to ip `10.0.0.154`
   - Those DNS names is available both in containers and on the node itself (kube-config automatically adds the info to `/etc/resolv.conf`)
   - If you want to access the Kubernetes API easily, `curl -k https://kubernetes` or `curl -k https://10.0.0.1` if you remember numbers better (`-k` stands for insecure as apiserver has no signed certs by default)
   - The DNS server itself has allocated ip `10.0.0.10`
   - The DNS domain is `cluster.local`
 - Central image registry
   - A registry for storing cluster images if e.g. the cluster has no internet connection for a while
   - Or for cluster-specific images that one not want to publish on Docker Hub
   - This service is available at this address: `registry.kube-system` when DNS is enabled
   - Just tag your image: `docker tag my-name/my-image registry.kube-system:5000/my-name/my-image`
   - And push it to the registry: `docker push registry.kube-system:5000/my-name/my-image`
  
`kube-ui` was removed, because the Kubernetes team shifted focus to [dashboard](https://github.com/kubernetes/dashboard).

## Service management

The `kube-systemd` rootfs uses systemd services for starting/stopping containers.

Systemd services: 
 - system-docker: Or `docker-bootstrap`. Used for running `etcd` and `flannel`.
 - etcd: Starts the `kubernetesonarm/etcd` container. Depends on `system-docker`.
 - flannel: Starts the `kubernetesonarm/flannel` container. Depends on `etcd`.
 - docker: Plain docker service. Dropins are symlinked. Depends on `flannel`.
 - k8s-master: Service that starts up the main master components
 - k8s-worker: Service that starts up `kubelet` and the `proxy`.


Useful commands for troubleshooting: 
 - `systemctl status (service)`: Get the status
 - `systemctl start (service)`: Start
 - `systemctl stop (service)`: Stop
 - `systemctl cat (service)`: See the `.service` files for an unit.
 - `journalctl -xe`: Get the system log
 - `journalctl -xeu (service)`: Get logs for a service

## Beta version

This project is under development.
[Changelog](CHANGELOG.md)

## Future work

See the [ROADMAP](ROADMAP.md)

## License

[MIT License](LICENSE)

## Goals for this project

This project ports [Kubernetes](https://github.com/kubernetes/kubernetes) to the ARM architecture. 
The primary boards used for testing is Raspberry Pi 2´s.

My goal for this project is that it should be as small as possible, while retaining its flexibility.  
It should also be as easy as possible for people, who don´t know anything about Kubernetes, to get started.

I also have opened a proposal for Kubernetes on ARM: [kubernetes/kubernetes#17981](https://github.com/kubernetes/kubernetes/issues/17981)

It should be easy in the future to add support for new boards and operating systems.

#### Feel free to create an issue if you find a bug or think that something should be added or removed!