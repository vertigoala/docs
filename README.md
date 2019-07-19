## Welcome to Vertigo ALA

This is a proof of concept for a dockerized version of a few ALA modules (Atlas of Living Australia). These repos will serve the purpose of setting a common ground for discussion among ALA mantainers.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### General steps for each module

- Fork the module
- Create a basic Dockerfile and implement a docker-compose workflow
- Implement console logging and verbosity
- Avoid code changes whenever possible
- Define a minimal set of externalized configs for the container
- Automate container build (Docker hub auto build and/or Gitlab CI)
- Create a sample project derived from this image (sample custom module)
- Create a helm chart for k8s deployment with self-documented values.yaml

### Understanding docker-compose workflow

Docker-compose is a tool that simplifies development of multi-container projects (pretty much all of them). That goes from the simple webapp + database case to multi-modules solutions like ALA itself.

The typical journey of any developer pulling the project for the first time is:

```sh
docker-compose up --build
```

This command will build the dockerfile and start the containers as dictated by `docker-compose.yml` (mounts, networks, service names, ports etc.). A developer can therefore make sure the project runs automagically in any case, regardless of developers' platform.

Further refinement can enable class hot-reloading, so that code maintenance is done over a live container.

### Localhost vs Remote deployment

All the default settings in docker-compose and properties files target localhost deployment in Docker Desktop developer workstations. The focus is on developer onboarding and predictability, not production deployments (for those we will provide helm charts).

As a middle ground we will try to maintain sample config and compose files for remote swarm mode deployment, testable on play-with-docker disposable nodes.

### Tips for remote play-with-docker deployment

[Play-with-docker](https://play-with-docker.com) is an excellent way to test a remote deployment for free.

1. Create PWD instances using the "3 Managers, 2 Workers" template
2. Write down manager1 external hostname using any of the tricks below, resulting in a hostname in the form "ipxxxxx-yyyyyyyyyy.direct.labs.play-with-docker.com":

    - Copy the SSH hostname PWD gives and replace "@" for "."
    - **OR** run the command below from manager1 shell for its output:

```sh
echo "ip$(ifconfig eth1 | grep Mask | awk '{print $2}'| cut -f2 -d: | tr '.' '-')-$SESSION_ID.direct.labs.play-with-docker.com"
```

3. Adjust your local DOCKER_HOST env var to the remote manager's engine in the form below, testing for the node list:

```sh
export DOCKER_HOST=tcp://<your-manager-external-hostname>:2375
docker node ls
ID                            HOSTNAME            ...
tew2q0h8s9eodehgf8qp8198t *   manager1            ...
hayzvxmep8gzh8vy0e6kxadrq     manager2            ...
7jopxdbgb44ulzrnt9zi8xkqh     manager3            ...
yihno6jr5uett2q9tedemoif4     worker1             ...
p20xoglmofu3id98ec6yd6kr0     worker2             ...
```

Congratulations, you are in control of a remote play-with-docker swarm from your local command-line promt.

Deployments against a remote swarm should de done like below:

```sh
cd <module>
docker stack deploy -c docker-compose-swarm.yml <module-name>
```

Removal is also done from the command-line:

```sh
cd <module>
docker stack remove <service-name>
```

### Implemented/deployed modules

- traefik
- commonui-bs3
- commonui-sample
- image-service
- image-sample

### Modules' comments

#### traefik

*Please ignore traefik if you are running the modules locally with docker-compose.*

Traefik serves as a http load balancer and reverse proxy for swarm mode deployment. In kubernetes traefik can also work as an ingress controller.

Problem is **YOU MUST** edit the `swarm/images-config.properties` file *before* deployment, so that the URL references are set to rely on traefik.

In swarm mode all modules will be available through traefik in port 8000 (as per docker-compose files), so you can edit `swarm/images-config.properties` entries to use any valid IP or DNS name.

Play-with-docker DNS names are random, but you can use a similar trick to the one you used before:

```sh
echo "ip$(ifconfig eth1 | grep Mask | awk '{print $2}'| cut -f2 -d: | tr '.' '-')-$SESSION_ID-8000.direct.labs.play-with-docker.com"
```

Use this hostname in `swarm/images-config.properties` when running remotely in play-with-docker and you will be fine.

Play-with-docker will also generate a clickable URL for the traefik container 8000 port, but the link itself is broken: you need to add the module path at its end ("/images" for image-service, for example).

#### commonui-bs3

This module gained a simple Dockerfile and a docker-compose workflow - it runs the same commonui-bs3 static web module that is live at "<https://www.ala.org.au/commonui-bs3>". It demonstrates the most basic techniques for a container project, but the produced image itself is almost useless: instead of running your own replica it is easier to just point to the live URL in your configs. When creating a custom commonui you are most likely to produce a static server image from scratch.

In brazilian ALA, for example, we built a single image with both commonui-bs3 and commonui-bs2 contexts - some modules use different commonui versions for static content (including javascript). Sweden bioatlas mounts a static local folder (containing bs2 and bs3 files) in a vanilla nginx image.

#### commonui-sample

We need self-contained images for a portable k8s deployment, but we can't be sure of what files must be hand-picked from the original commonui-* projects. In order to keep this project small the entire commonui-bs3 and commonui-bs2 content is downloaded at build time (see Dockerfile), but overwritten by this project's files. This reduces the amount of mantained files.

Docker-compose workflow makes an interesting trick replacing the default command in a developer's machine: the versioned pages are mounted in a staging folder and copied into the web server folder every 2 seconds. Kinda lame but it works.

- Home at <http://localhost:8000/commonui-bs3/>

#### image-service

This is the image-service module image packaged as a reusable asset for any ALA deployment.

Again docker-compose is used for local build and development. The image contains a minimal `image-config.properties` that defaults to:

- Australia CAS login
- Bypass CAS security
- Postgres database at jdbc:postgresql://pgdbimage/images (side container)
- Elasticsearch running on a side container
- Home at <http://localhost:8080/images/>
- Admin at <http://localhost:8080/images/admin>
- Tomcat itself playing as image-store at <http://localhost:8080/store>

Bypassing security with CAS enabled is required for admin pages to work for any user for most ALA modules (good for local development).

#### image-sample

This is a sample deployment of the reusable image-service container with a custom properties file. A custom commonui runs in a side container and image-store is played by a dedicated nginx container.

Again docker-compose is used for local build and testing. The image contains a minimal `image-config.properties` that defaults to:

- Australia CAS login
- Bypass CAS security
- Postgres database at jdbc:postgresql://pgdbimage/images (side container)
- Elasticsearch running on a side container
- Commonui runs in a side container (`vertigoala/commonui-sample`) at <http://localhost:8000/commonui-bs3>
- Home at <http://localhost:8080/images/>
- Admin at <http://localhost:8080/images/admin>
- Image-store at <http://localhost:8001/store>

An alternative sample properties for remote swarm mode deployment is mantained in the "swarm" folder. Every hostname should be replaced by a valid swarm load balancer (in case of PWD any node's external hostname will do). A specific docker-compose is kept in `docker-compose-swarm.yml` because `docker stack deploy` commands require a little different syntax.
