# Intention

This repository is intended to hold artifacts for creating RHEL Image Mode images 
that will be used to image edge devices managed by FlightCtl (aka Red Hat Edge Manager).
The repo also contains artifacts for creating a FlightCtl server (both connected and airgapped).
At the moment, the files contained in this repository are a bit haphazard and disorganized
but this will be fixed in the near term.

# FlightCtl

Content TBD (from https://github.com/flightctl/flightctl/blob/main/docs/user/introduction.md)

# RHEL Image Mode
RHEL Image Mode enables management of edge device systems in a simplified
way by taking advantage of container application infrastructure. By
packaging operating system updates as an OCI container, RHEL Image Mode
simplifies the distribution and deployment of operating systems and
their updates, easing the amount of resources necessary to maintain a
disparate fleet of edge devices.


## Setup
Start with a minimal install of RHEL 9.6 either on baremetal or on a guest
VM. Use UEFI firmware, if able to, when installing your system. Also make
sure there's sufficient disk space on the RHEL 9.4 instance to support the
demo. I typically configure a 128 GiB disk on the guest VM.  During RHEL
installation, configure a regular user with `sudo` privileges on the host.

These instructions assume that this repository is cloned or copied to your
user's home directory on the host (e.g. `~/rhel-bootc-image-gen`). The
instructions below follow that assumption.

Login to the host and then run the following commands to create an SSH
keypair that you'll use later to access the edge device. Even though you
really should set a passphrase, skip that when prompted to make the demo
a little easier to run.

    cd ~/rhel-bootc-image-gen
    ssh-keygen -t rsa -f ~/.ssh/id_core

Edit the `demo.conf` file and make sure the settings are correct. At a
minimum, you should adjust the credentials for simple content access.
The full list of options in the `demo.conf` file are shown here.

This repo includes content to create a local image repository to serve RHEL image mode
images.  If you prefer to use another image repository, please ignore all references
to "local container registry."

| Option           | Description |
| -----------------| ----------- |
| SCA_USER         | Your username for Red Hat Simple Content Access |
| SCA_PASS         | Your password for Red Hat Simple Content Access |
| EPEL_URL         | The Extra Packages for Enterprise Linux URL |
| EDGE_USER        | The name of a user on the target edge device |
| EDGE_PASS        | The plaintext password for the user on the target edge device |
| BOOT_ISO         | Minimal boot ISO used to create a custom ISO with a custom kickstart file |
| EDGE_HASH        | A SHA-512 hash of the EDGE_PASS parameter |
| SSH_PUB_KEY      | The SSH public key of a user on the target edge device |
| HOSTIP           | The IP address of the local container registry - if relevant|
| REGISTRYPORT     | The port for the local container registry - if relevant |
| CONTAINER_REPO   | The fully qualified name for your bootable container repository |
| REGISTRYINSECURE | Boolean for whether the registry requires TLS |

Make sure to download the RHEL 9.6 `BOOT_ISO` file, e.g. [rhel-9.6-x86_64-boot.iso](https://access.redhat.com/downloads/content/rhel)
to the local copy of this repository on your RHEL instance
(e.g. ~/rhel-bootc-image-gen).

Run the following script to register with Red Hat and update the system.

    sudo ./register-and-update.sh
    sudo reboot

After the system reboots, run the following script to install container
and ISO image tools.

    cd ~/rhel-bootc-image-gen
    sudo ./config-bootc.sh

You can use a publicly accessible registry like [Quay](https://quay.io)
but if you want to run this demo disconnected, you can also optionally
set up a local container registry using the following script.

    cd ~/rhel-bootc-image-gen
    sudo ./config-registry.sh

NB: If you set up an insecure registry on another RHEL instance,
please make sure to copy the `999-local-registry.conf` file to the
`~/rhel-bootc-image-gen` and `/etc/containers/registries.conf.d`
directories on this RHEL instance that will build the bootable container
images.

Login to Red Hat's container registry using your Red Hat customer portal
credentials and then pull the container image for the base bootable
container.

    podman login registry.redhat.io
    podman pull registry.redhat.io/rhel9/rhel-bootc:9.6

At this point, setup is complete.

## Build the base container image
Use the following command to build the `base` bootable container
image. This image contains the Firefox browser running in kiosk mode.

    cd ~/rhel-bootc-image-gen
    . demo.conf
    podman build -f BaseContainerfile -t $CONTAINER_REPO:base \
        --build-arg DEMO_USER=$DEMO_USER

Push the image to the registry.

    podman push $CONTAINER_REPO:base


## Deploy the image using an ISO file
Run the following command to generate an installable ISO file for your
bootable container. This command prepares a kickstart file to pull
the bootable container image from the registry and install that to the
filesystem on the target system. This kickstart file is then injected
into the standard RHEL boot ISO you downloaded earlier. It's important to
note that the content for the target system is actually in the bootable
container image in the registry. This ISO merely contains enough to start
the system and then use the kickstart file to pull the operating system
content from the container registry.

    sudo ./gen-iso.sh

The generated file is named `bootc-rhel.iso`. Use that file to boot
a physical edge device or virtual guest. Ensure that you use the UEFI
firmware option for a virtual guest or install to a physical edge device
that supports UEFI. Make sure this system is able to access your public
registry to pull down the bootable container image.

You may see a core dump when running in a guest VM if
memory is low. I've successfully tested running the bootable containers
on a laptop with 16GB of memory and a 512GB SDD.

Test the deployment by verifying that the kiosk user automatically logs
into a desktop where only the web browser is available with no other
desktop controls.
