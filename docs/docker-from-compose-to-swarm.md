---
tags: Docker, Blog
---

> This page is WIP - as I progress through the journey, I'll update my notes and this page will be refreshed a couple minutes later

# How to migrate a docker compose file to Docker Swarm Mode?
...from `docker compose up -d` to `docker stack deploy --compose-file docker-compose.yaml ha`

## 0. Backstory: Home Assistant in a container

Boring story, for context:

>*After installing a PKI and OpenVPN manually on a Raspberry Pi 4, I've been wanting a more flexible way to install/remove/update stuff on it... Containers looked like the way to go... but what if I want to run many things? That'd be fun to have a cluster of Raspberry Pi... which led me to find the Turing Pi 2 cluster board. And as I researched what people usually deploy on Raspberry clusters, I kept seeing the Home Assistant logo...*

>*So I looked at it up close and started running a Home Assistant instance.
While waiting for the Turing Pi 2 to release, I thought I would run the containers directly on my desktop using Docker Compose. And if I did it right, I would "just" have to move my containers to the new hardware once ready, right?... well that takes a couple precautions*

### A first docker compose, running on windows / docker desktop 

So I started creating a docker compose file, with just Home Assistant to begin with.
Simple container, local bind
Adding USB... Leading to USBIP
Then needing to install a distro in the WSL2 (why?)

### Adding more container to compose... and more complexity

Architecture diagramme
Specific features used

### Thinking of the migration to a cluster... (too soon to dive in this?)

Will have to install docker and configure many compute modules. Don't really want to do that manually and miss anything... Here comes Ansible 
TODO: ansible for setting up the Swarm Mode

## 1. Looking at Swarm Mode: many questions came up...

That's probably what I googled when thinking of the migration to Swarm Mode:
- how to deploy a **compose file** in swarm mode?
- how to access **bound folders** in swarm mode?
- how to access **USB devices** in swarm mode?
- how to **restart** malfunctioning containers/services? (like self-healing from k8s)

Didn't find very direct answers so I digged up a bit more...

### What needs to be changed in my compose file to migrate so swarm mode?

Impact of swarm mode on usual sections from compose
This is not a comprehensive list (refer to the compose reference for that), but that's all the things I had to tinker with in my compose file to make it swarm friendly:

- ``privileged: true`` 
    - Ignored in swarm. You should not need it 
- ``restart: always``
    - Ignored in Swarm. Not an issue as swarm services will be destroyed / recreated as necessary (to reach the constraints)
- ``container_name:`` 
    - Ignored in swarm. Doesn't really matter anyway?
- ``device:`` 
    - Ignored in swarm. Ok this one is an issue, particularly with our USB devices needed to communicate with the real world (Zigbee bridge, RFlink or Coral accelerator etc.)
- ``volume / binds (inline):`` 
    - Similar to the USB devices, you can't mount bind on a node, while services are moving potentially on any node of the cluster.
- ``shm_size:``
    - https://docs.frigate.video/installation/#calculating-required-shm-size
    - TODO: ?? what impacts on Frigate --> workaround with a volume tmpfs targeted to /dev/shm

### How to pass devices in Swarm Mode?

Two things to deal with:

1. By construction, services in a swarm can be run on any node... which will not work if we need to access a specific USB device. Can't wait to be lucky and be on the right node
    - For this we are going to use the swarm specific section "deploy" to indicate a constraints for the service needing the USB device.
    
    ```
    (FIXME: example of deploy section)
    ```

2. Even on the right node, how can we pass the USB device... without the "device" section?
    - This one is trickier, we have to pass the device as a **volume**, and manually authorise the device for the container using a number of scripts
    - TODO: Continue the explanation of the different scripts: udev rule, find the container, create the authorization manually


### Volumes and storage

[Volumes in swarm](https://stackoverflow.com/questions/55288453/docker-volume-in-swarm)

>A volume is a way for docker to describe a mount point. When a volume get created, it doesn't actually get physically mounted anywhere until a container needs it.
>So if you have a docker swarm and multiple nodes, when you create a volume, essentially the description of the volume gets replicated on each nodes but nothing else happens.
>When a container boot up, it will try to mount a volume on the host its being booted up. If the volume wasn't physically present it will get mounted/created for the first time and reused there. So if you're using the local driver, it will essentially create the folder and that's it.
>If you have multiple hosts, it means each host will create its own folder on demand.

  - TODO: use NFS volumes

## 2. How to actually use the updated compose file in Swarm Mode? Testing...

Can actually switch to swarm, test deploying a stack -- and if it doesn't work, remove it and put your compose back up. No need to exit/delete the swarm

- Comment the network part of docker compose file
- Then:
  ```
  docker stack deploy --compose-file docker-compose.yaml homeassistant
  ```
- At the end:
  ```
  docker stack rm homeassistant
  ```
- Uncomment the network section


<figure markdown>

### Test image inclusion + lightbox

<figure>

![Diablo IV Early access Wiz](./images/wizard.png) 

<figcaption markdown> Diablo IV Early access Wiz. Credit: [me](https://twitter.com/poulti) </figcaption>

</figure>