---
title: "Podman"
date: 2021-02-03T13:14:50+07:00
descriptiom: General introduction to Podman 
tags: ["Podman","container","automation"]
---
## What is Podman

Podman is a a daemonless, open-source tool to manage, deploy and build application containers. Podman usese Open Containers Initiative (OCI) Containers and container images, which mean that containers crated with/for Docker or CRI-o will work with Podman as well and vice versa. It is coming with a command line interface (CLI) which pretty much offers the same commands like Docker does. How similar are podman commands to docker's? Let's put it this way: many Podman just alias `docker` to `podman`. Like other runtimes, podman also relies on an OCI comliant container runtime to interface with the operating system. Podamn also has a RESTful API to manage containers, offers automatic updates of containers and has a fantastic `systemd` integration.

## Why Podman?

Due to a number of issues around Docker, the way they interact and treat the community and the number of ignored security issues my choice of container management software falls on Podman. Podman has the same and in many cases better abilities than Docker. Podman supports `pods` which can be implemented on a standalone server and moved to others or even deployed in K8S.

## Before and after (a recap)

Previously we had everything running in KVM VMs. This required over 200GB of ram and 32 cores over 2 Xeon Silver CPUs. Expensive. With choice of Podman and containerization of our infra we reduced that hardware requirement to about 10% of it. This is able to sustain almost all of our requirements in a containerized format at a significantly lower TCO.

The barrier of entry for containers is low and to make thing even easier I use openSUSE MicroOS as the host of my podman services. However, keeping the containers up-to-date and running 24/7 requires some deeper understanding and knowledge. 

## Getting containers running

### Registries 

Podman supports every OCI compliant registries like Dockerhub, Quay.io, Treescale etc. To check, add, and remove the registries the system uses one can simply edit `/etc/containers/registries.conf`.
```
# For more information on this configuration file, see containers-registries.conf(5).
#
# Registries to search for images that are not fully-qualified.
# i.e. foobar.com/my_image:latest vs my_image:latest
[registries.search]
registries = ["registry.opensuse.org", "docker.io"]

# Registries that do not use TLS when pulling images or uses self-signed
# certificates.
[registries.insecure]
registries = []

# Blocked Registries, blocks the `docker daemon` from pulling from the blocked registry.  If you specify
# "*", then the docker daemon will only be allowed to pull from registries listed above in the search
# registries.  Blocked Registries is deprecated because other container runtimes and tools will not use it.
# It is recommended that you use the trust policy file /etc/containers/policy.json to control which
# registries you want to allow users to pull and push from.  policy.json gives greater flexibility, and
# supports all container runtimes and tools including the docker daemon, cri-o, buildah ...
[registries.block]
registries = []
```
As I'm using openSUSE I do have `registry.opensuse.org` added to my registries. You can add for example `quay.io` to this list to be able to search, pull, and push to Quay.

#### Login to registry

Just like with Docker you can log in to a registry with your username and password using `podman login [registry]`. Once logged in the credentials will be stored in `${XDG_RUNTIME_DIR}/containers/auth.json`. For the sake of automation you could make a copy of this `auth.json` file to be used in different scripts for building or deploying containerized applications, but keep the file with your credentials safe.

### Searching for a container

To search with podman for an application use:
```
$ podman search nginx                                                                                                                                                     
INDEX         NAME                                                                                                                  DESCRIPTION                                      STARS   OFFICIAL  AUTOMATED
opensuse.org  registry.opensuse.org/devel/kubic/containers/container/opensuse/nginx                                                                                                  0                 
opensuse.org  registry.opensuse.org/home/dcassany/obs-service/container/kubic-nginx-ingress-controller                                                                               0                 
opensuse.org  registry.opensuse.org/home/dcassany/obs-service/container/kubic/nginx-ingress-controller                                                                               0                 
opensuse.org  registry.opensuse.org/home/dmolkentin/video/containers/opensuse-nginx                                                                                                  0                 
opensuse.org  registry.opensuse.org/home/illuusio/images/images_15.2/opensuse/ilmi/nginx-fpm                                                                                         0                 
opensuse.org  registry.opensuse.org/home/illuusio/images/images_15.2/opensuse/ilmi/nginx-ssl                                                                                         0                 
opensuse.org  registry.opensuse.org/home/illuusio/images/images_15.2/opensuse/ilmi/nginx-uwsgi                                                                                       0                 
opensuse.org  registry.opensuse.org/home/mook_work/scf/nginx-buildpack-release/images/images/nginx-buildpack                                                                         0                 
opensuse.org  registry.opensuse.org/home/onielsen/virtualization/kubic-container/15.0/containers/kubic/nginx                                                                         0                 
opensuse.org  registry.opensuse.org/home/pgeorgiadis/branches/devel/caasp/kubic-container/container/kubic/nginx-ingress-controller                                                   0                 
opensuse.org  registry.opensuse.org/home/sjamgade/helm/opensuse_leap_15.2/nginx                                                                                                      0                 
opensuse.org  registry.opensuse.org/opensuse/factory/arm/totest/containers/opensuse/nginx                                                                                            0                 
opensuse.org  registry.opensuse.org/opensuse/factory/powerpc/totest/containers/opensuse/nginx                                                                                        0                 
opensuse.org  registry.opensuse.org/opensuse/factory/powerpc/totest/images/opensuse/nginx                                                                                            0                 
opensuse.org  registry.opensuse.org/opensuse/factory/totest/containers/opensuse/nginx                                                                                                0                 
opensuse.org  registry.opensuse.org/opensuse/factory/zsystems/images/opensuse/nginx                                                                                                  0                 
opensuse.org  registry.opensuse.org/opensuse/factory/zsystems/totest/containers/opensuse/nginx                                                                                       0                 
opensuse.org  registry.opensuse.org/opensuse/nginx                                                                                                                                   0                 
docker.io     docker.io/library/nginx                                                                                               Official build of Nginx.                         14378   [OK]      
docker.io     docker.io/jwilder/nginx-proxy                                                                                         Automated Nginx reverse proxy for docker con...  1951              [OK]
```

To search in a spcific repository:
```
$ podman search registry.opensuse.org/nginx                                                                                                                               
INDEX         NAME                                                                                                                  DESCRIPTION  STARS   OFFICIAL  AUTOMATED
opensuse.org  registry.opensuse.org/devel/kubic/containers/container/opensuse/nginx                                                              0                 
opensuse.org  registry.opensuse.org/home/dcassany/obs-service/container/kubic-nginx-ingress-controller                                           0                 
opensuse.org  registry.opensuse.org/home/dcassany/obs-service/container/kubic/nginx-ingress-controller                                           0                 
opensuse.org  registry.opensuse.org/home/dmolkentin/video/containers/opensuse-nginx                                                              0                 
opensuse.org  registry.opensuse.org/home/illuusio/images/images_15.2/opensuse/ilmi/nginx-fpm                                                     0                 
opensuse.org  registry.opensuse.org/home/illuusio/images/images_15.2/opensuse/ilmi/nginx-ssl                                                     0                 
opensuse.org  registry.opensuse.org/home/illuusio/images/images_15.2/opensuse/ilmi/nginx-uwsgi                                                   0                 
opensuse.org  registry.opensuse.org/home/mook_work/scf/nginx-buildpack-release/images/images/nginx-buildpack                                     0                 
opensuse.org  registry.opensuse.org/home/onielsen/virtualization/kubic-container/15.0/containers/kubic/nginx                                     0                 
opensuse.org  registry.opensuse.org/home/pgeorgiadis/branches/devel/caasp/kubic-container/container/kubic/nginx-ingress-controller               0                 
opensuse.org  registry.opensuse.org/home/sjamgade/helm/opensuse_leap_15.2/nginx                                                                  0                 
opensuse.org  registry.opensuse.org/opensuse/factory/arm/totest/containers/opensuse/nginx                                                        0                 
opensuse.org  registry.opensuse.org/opensuse/factory/powerpc/totest/containers/opensuse/nginx                                                    0                 
opensuse.org  registry.opensuse.org/opensuse/factory/powerpc/totest/images/opensuse/nginx                                                        0                 
opensuse.org  registry.opensuse.org/opensuse/factory/totest/containers/opensuse/nginx                                                            0                 
opensuse.org  registry.opensuse.org/opensuse/factory/zsystems/images/opensuse/nginx                                                              0                 
opensuse.org  registry.opensuse.org/opensuse/factory/zsystems/totest/containers/opensuse/nginx                                                   0                 
opensuse.org  registry.opensuse.org/opensuse/nginx                                                                                               0                 
```

## Pulling a container

To pull a container:
```
$ podman pull nginx                                                                                                                                                      Completed short name "nginx" with unqualified-search registries (origin: /etc/containers/registries.conf)
Trying to pull registry.opensuse.org/nginx:latest...
  name unknown
Trying to pull docker.io/library/nginx:latest...
Getting image source signatures
Copying blob f72584a26f32 done  
Copying blob 0732ab25fa22 done  
Copying blob 7125e4df9063 done  
Copying blob a076a628af6f done  
Copying blob d7f36f6fe38f done  
Copying config f6d0b4767a done  
Writing manifest to image destination
Storing signatures
f6d0b4767a6c466c178bf718f99bea0d3742b26679081e52dbf8e0c7c4c42d74
```
To pull from a specific registry:
```
$ podman pull registry.opensuse.org/opensuse/nginx                                                                                                                       
Trying to pull registry.opensuse.org/opensuse/nginx:latest...
Getting image source signatures
Copying blob fc491b4cab32 done  
Copying blob fd4952530c86 done  
Copying config 36fa073c57 done  
Writing manifest to image destination
Storing signatures
36fa073c57cc5251ac2df80de6a2e052dad9a10d6ffc9a9e65ba6eaa0ddea6ff
```
To pull from a registry authenticated with an `auth.json` file stored in `~/.secret`:
```
$ podman pull --authfile=~/.secret/auth.json registry.opensuse.org/opensuse/nginx                                                                                
Trying to pull registry.opensuse.org/opensuse/nginx:latest...
Getting image source signatures
Copying blob fc491b4cab32 done  
Copying blob fd4952530c86 done  
Copying config 36fa073c57 done  
Writing manifest to image destination
Storing signatures
19162aa1f28d9b92c87f788939bad1e3f6100ac8a1aeeed9448f642e8eb695d2
```
### Starting a container

#### Rootfull or rootless
As Podman is daemonless running containers doesn't require running them with root, but can be started with a user as well. Granted, rootless containers will have no access to networking and can't use priviliged ports, but it is a great way to keep the system secure. Rootfull containers are recommended for running ceratin applications that require access over priviliged ports like an nginx reverse proxy.

Starting a container that runs in the background and exposing port 8000 to be accessible from the outside:
```
$ podman run -d -p 8000:8000 nginx                                                                                                      
41e3829d1c800d40feb9d88ac962d29c8256b60ec5747f732717893d93b0f35e
```
To list running containers:
```
[~] podman ps                                                                                                                                              
CONTAINER ID  IMAGE                           COMMAND               CREATED        STATUS            PORTS                   NAMES
41e3829d1c80  docker.io/library/nginx:latest  nginx -g daemon o...  3 seconds ago  Up 3 seconds ago  0.0.0.0:8000->8000/tcp  suspicious_feistel
```
This list provides a lot of information about running containers. 
To list every container, running or not:
```
$ podman ps -a
CONTAINER ID  IMAGE                           COMMAND               CREATED         STATUS                      PORTS                   NAMES
81a872daad56  docker.io/library/nginx:latest  nix                   6 seconds ago   Exited (127) 6 seconds ago  0.0.0.0:8001->8001/tcp  wonderful_rubin
41e3829d1c80  docker.io/library/nginx:latest  nginx -g daemon o...  18 minutes ago  Up 18 minutes ago           0.0.0.0:8000->8000/tcp  suspicious_feistel
bc5030bffd6c  docker.io/library/nginx:latest  nginx -g daemon o...  18 minutes ago  Created                     0.0.0.0:80->8000/tcp    jolly_johnson
```
If needed the output format of `podman ps` can be manipulated with the `--format` flag. For example to list only the container ID and the image name:
```
[~] podman ps --format "{{.ID}} {{.Image}}"                                                                                                                           
41e3829d1c80 docker.io/library/nginx:latest
```
This pretty print containers to JSON. The following placeholders are available:
* .ID: Container ID                             
* .Image:Image Name/ID
* .ImageID: Image ID
* .Command: Quoted command used                      
* .CreatedAt: Creation time for container              
* .RunningFor: Time elapsed since container was started 
* .Status: Status of container
* .Pod: Pod the container is associated with     
* .Ports: Exposed ports                            
* .Size: Size of container
* .Names: Name of container                        
* .Labels: All the labels assigned to the container
* .Mounts: Volumes mounted in the container            
For more visit the `podman ps` man page.

#### Deleting containers and images
Running containers or images that are being used can't be deleted unless it is forced. The best course of action is to stop the container then delete it, and then delete the image.
```
[~] podman ps #List running containers                                                                                                                          
CONTAINER ID  IMAGE                           COMMAND               CREATED         STATUS             PORTS                   NAMES
41e3829d1c80  docker.io/library/nginx:latest  nginx -g daemon o...  22 minutes ago  Up 22 minutes ago  0.0.0.0:8000->8000/tcp  suspicious_feistel
$ podman stop suspicious_feistel #stop container by name                                                                             
41e3829d1c800d40feb9d88ac962d29c8256b60ec5747f732717893d93b0f35e
$ podman rm suspicious_feistel #Delete container by name
41e3829d1c800d40feb9d88ac962d29c8256b60ec5747f732717893d93b0f35e
[~] podman rmi nginx #Delete container image, this fails as there are other containers still using it                      
Error: 1 error occurred:
        * could not remove image f6d0b4767a6c466c178bf718f99bea0d3742b26679081e52dbf8e0c7c4c42d74 as it is being used by 2 containers: image is being used
[~] podman rm jolly_johnson wonderful_rubin #Remove containers that are still using the nginx image                                    
81a872daad5661a7704e47c4f5025db502852cf7cc0aaaa439e71a9fa915e9b9
bc5030bffd6c604192c31911141d728378b69832cafbf81d97d976d59fc5f86d
[~] podman rmi nginx #Delete the image                                                                         
Untagged: docker.io/library/nginx:latest
Deleted: f6d0b4767a6c466c178bf718f99bea0d3742b26679081e52dbf8e0c7c4c42d74
$ podman images #List available, pulled images
REPOSITORY                                 TAG     IMAGE ID      CREATED      SIZE
registry.opensuse.org/opensuse/tumbleweed  latest  111d1befeae4  2 days ago   94.4 MB
registry.opensuse.org/opensuse/nginx       latest  36fa073c57cc  3 days ago   103 MB
docker.io/prom/prometheus                  latest  19162aa1f28d  2 weeks ago  174 MB
```

#### Managing images
To list all available images use `podman images`. This will provide a list of every image that is currently available in the system and ready to be used without needing to pull it from a registry. To delete an image use `podman rmi`. Visit the `podman images` man page for more.

### Pod creation

We create the pods by naming them - duh - and exposing the required ports. This way we're not required to setup containers individually in terms of publishing ports just adding the containers to them.   

#### Pod creation:
```
podman pod create -n monitor -p 9090:9090 -p 3000:3000
podman pod create --name sx5 -p 3306:3306 -p 8888:80 -p 8843:443 -p 6379:6379 -p 9980:9980
```

#### Adding containers to the pods:

##### Monitoring:
```
# podman run -d --pod monitor --name prometheus --label io.containers.autoupdate=image -v /home/podman_vol/prometheus/etc-prometheus/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
# podman run -d --pod monitor --label io.containers.autoupdate=image --mount type=bind,source=/home/podman_vol/grafana/var-lib-grafana,target=/var/lib/grafana --name grafana grafana/grafana:latest
```
##### SX5:
```
# podman run -d --label io.containers.autoupdate=image --pod sx5 --name sx5-www --mount type=bind,source=/home/podman_vol/sx5/sx5_data,target=/home/sx5_data --mount type=bind,source=/home/podman_vol/sx5/www/sx5,target=/srv/www/htdocs/sx5 registry.s8i.io/sx5-www
# podman run -t -d --pod sx5 --label io.containers.autoupdate=image -e 'domain=reno\\.s8i\\.io' -e 'username=apinter' -e 'password=t43heta@3861' --name collabora --restart always --cap-add MKNOD collabora/code
# podman run -d --label io.containers.autoupdate=image --pod sx5 --name sx5-db --mount type=bind,source=/home/podman_vol/sx5/sql/var-lib-mysql,target=/var/lib/mysql registry.s8i.io/mariadb-sx5
# podman run --label io.containers.autoupdate=image --pod sx5 --name redis -d redis
```

### Container registry

#### Podman auto-update
Podman's other important feature is the automatic updates of containers without the need of even putting containers in pods or adding a watchtower service. This is done by adding the ```--label io.containers.autoupdate=image```  during container creation.    
This to be followed by generating systemd units for the container: ```podman generate systemd --new --files [container ID or name]```. Once this is ready and the unit file is under the corresponding systemd folder - depending on how podman is used w/ root or rootless - the container unit to be started. Following this the ```podman auto-update``` command can update every container with changed images.

#### Podman updater script
We're shipping a script that every 2 and 4am runs, pulls the images and updates every container. This ran by a service unit that is triggered by a timer. !!! THIS MIGHT CHANGE LATER !!!!
``` bash
#!/bin/bash

build_www(){
cd /home/podman_vol/dev/sx5/www; \
podman build -t sx5-www .
podman push --authfile /root/.auth/auth.json sx5-www registry.s8i.io/transmission-app
}

build_db(){
cd /home/podman_vol/dev/sx5/sql; \
podman build -t mariadb-sx5 .
podman push --authfile /root/.auth/auth.json mariadb-sx5 registry.s8i.io/mariadb-sx5 
}

build_tr(){
cd /home/podman_vol/dev/transmission; \
podman build -t transmission-app .
podman push --authfile /root/.auth/auth.json transmission-app registry.s8i.io/transmission-app
}

auto_update(){
podman auto-update
}

pull;
build_www;
build_db;
build_tr;
auto_update;
```

