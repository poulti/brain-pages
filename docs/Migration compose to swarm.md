---
tags: Docker, Blog
---

# How to migrate a docker compose file to swarm mode?
...from `docker compose up -d` to `docker stack deploy --compose-file docker-compose.yaml ha`

## Backstory: it's a long story... A Home Assistant journey
After installing a PKI and Open on manually,  I've been wanting a more flexible way to install / remove and update stuff on my raspberry pi... Containers looked like the perfect solution - which led me to find the Turing Pi 2 cluster board. And kept seeing the Home Assistant logo as an example of stuff you could run on home labs...
So I looked at it up close, and while waiting for the Turing Pi 2 to release, thought I would give it a try and run the containers directly on my desktop. And if I did it right, I would "just" have to move my containers to the new hardware once ready... So I started creating a docker compose file, with just Home Assistant to begin with.

## A first docker compose, running on windows / docker desktop 

Simple container, local bind
Adding USB... Leading to USBIP
Then needing to install a distro in the WSL2

## Thinking of the migration to a cluster...

Will have to install docker and configure many compute modules. Don't really want to do that manually and miss anything... Here comes Ansible 

## List of questions I've asked myself... Maybe that can help you?
- how to deploy a **compose file** in swarm mode?
- how to access **bound folders** in swarm mode?
- how to access **USB devices** in swarm mode?
- how to **restart** malfunctioning containers/services? (like self-healing from kubernetes k8s)

## What needs to be changed in my compose file to migrate so swarm mode?
#### Impact of swarm mode on usual sections from compose

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
  - ?? what impacts on Frigate --> workaround with a volume tmpfs targeted to /dev/shm

## How to pass devices in swarm mode?

Two things to deal with: 
1. By construction, services in a swarm can be run on any node... which will not work if we need to access a specific USB device. Can't wait to be lucky and be on the right node
   - For this we are going to use the swarm specific section "deploy" to indicate a constraints for the service needing the USB device.
2. Even on the right node, how can we pass the USB device... without the "device" section?
   - This one is trickier, we have to pass the device as a **volume**, and manually authorise the device for the container using a number of scripts

## Volumes and storage

https://stackoverflow.com/questions/55288453/docker-volume-in-swarm
A volume is a way for docker to describe a mount point. When a volume get created, it doesn't actually get physically mounted anywhere until a container needs it.
So if you have a docker swarm and multiple nodes, when you create a volume, essentially the description of the volume gets replicated on each nodes but nothing else happens.
When a container boot up, it will try to mount a volume on the host its being booted up. If the volume wasn't physically present it will get mounted/created for the first time and reused there. So if you're using the local driver, it will essentially create the folder and that's it.
If you have multiple hosts, it means each host will create its own folder on demand.


## What can you test before migrating to a real swarm cluster?

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