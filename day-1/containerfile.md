## 2. Create Custom Images

In this lab we are going to see how an image is built:

 * An image is specified on a file called _Containerfile_ or _Dockerfile_ 
 * An image bases on a _base image_
 * A base image provides a common basis for building an image
 * Each image is composed by layers
 * Each instruction on the Containerfile adds a layer

_NOTE for us (to be removed): we neeed to use docker.io images in all examples with podman, otherwise each participant need a login to redhat. I guess that's the reason the original garage workshop only features oc commands and no podman part. When working with the openshift cluster, the access to the registry is already solved._

This is an example of a Dockerfile:
```
FROM redhat/ubi8
# The FROM command tell us which is the base image. We are using a dockerhub image published by RedHat. 
# https://hub.docker.com/r/redhat/ubi8

MAINTAINER eramon
# The MAINTAINER command tells who is the author of the Containerfile

LABEL description="My very own Apache Server"
# The LABEL is optional but useful to provide information about the image

RUN yum install -y httpd &&\
    yum clean all
# The RUN command is executed during build
# Each RUN command adds a layer to the image, so it's a good practice to use && to combine commands 

EXPOSE 80
# The EXPOSE command says which port on the container should be exposed (metadata only)

ENTRYPOINT ["httpd"]
# The ENTRYPOINT is the command to be executed on the container when it is started 
# The entrypoint can't be overriden

CMD ["-D", "FOREGROUND"]
# The CMD command provides the parameters for the command specified in ENTRYPOINT
# The arguments can be overriden by passing them at the end of podman run
```

_NOTE: Notice that we talk from a Dockerfile or Contaierfile arbitrarly. Both names are interchangeable and accepted by podman._

Explanations to the different commands are included in the file as comments.

Our Containerfile is ready - let's build the image with _podman build_:
```
eramon:cloud-native-labs$ podman build -t myapache:0.1 .
STEP 1: FROM redhat/ubi8-minimal
✔ docker.io/redhat/ubi8-minimal:latest
Trying to pull docker.io/redhat/ubi8-minimal:latest...
...
Successfully tagged localhost/myapache:0.1
8997b9661025a5baa1b0fc5c4dfc73508ff32eb26afaa34ce752098c6f7fd58b
```

In the list of images, we see both the image we created and the base image:
```
eramon:cloud-native-labs$ podman images
REPOSITORY             TAG         IMAGE ID      CREATED        SIZE
localhost/myapache     0.1         8997b9661025  2 minutes ago  292 MB
docker.io/redhat/ubi8  latest      cc0656847854  3 weeks ago    235 MB
```

Run a container from this image:
```
eramon:cloud-native-labs$ podman run --name myapache -d -p 8080:80 myapache
278e7d99275e981315f06c5517847d810e80b6153eaed30a29134106771129b7
eramon:cloud-native-labs$ podman ps 
CONTAINER ID  IMAGE                   COMMAND        CREATED        STATUS            PORTS                 NAMES
278e7d99275e  localhost/myapache:0.1  -D FOREGROUND  3 seconds ago  Up 3 seconds ago  0.0.0.0:8080->80/tcp  myapache
```

We created a new container based on our custom image, with name _myapache_, in dettached mode and forwarding port 8080 on the host to port 80 on the container. 

Should we try it?
```
eramon:cloud-native-labs$ curl localhost:8080
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN" "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd">
...
		<title>Test Page for the HTTP Server on Red Hat Enterprise Linux</title>
...
```

It looks like it's working :)

We saw in the Containerfile that there is an ENTRYPOINT and a CMD commands. Both commands are similar but there is an important difference:

 * Both ENTRYPOINT and CMD are executed when the container is created 
 * ENTRYPOINT is the command and CMD are the parameters, appended at the end
 * You can override the CMD with _podman run_ passing the parameters when running _podman run_
 * You can't override the entrypoint: instead anything added to the end of _podman run_ is appended to the command

Let's try overriding CMD:
```
eramon:cloud-native-labs$ podman run --name override -p 8081:80 myapache -v
Server version: Apache/2.4.37 (Red Hat Enterprise Linux)
Server built:   Oct 26 2021 14:18:06
```

What did we do? Instead starting the web server by executing _httpd -d foreground_ we printed out the apache version by calling _httpd -v_. The version was printed and the container finished, so the container is not running anymore:
```
eramon:cloud-native-labs$ podman ps -a
CONTAINER ID  IMAGE                   COMMAND        CREATED         STATUS                    PORTS                 NAMES
...
e42a2930ed34  localhost/myapache:0.1  -v             2 minutes ago   Exited (0) 2 minutes ago  0.0.0.0:8081->80/tcp  override
```

To finish a little housekeeping:
```
eramon:cloud-native-labs$ podman stop --all
ac05d949910abc9ffe04ed8d34d2a4c2024de13ae234dc16233a92b2c3c9250f
eramon:cloud-native-labs$ podman rm --all
ac05d949910abc9ffe04ed8d34d2a4c2024de13ae234dc16233a92b2c3c9250f
eramon:cloud-native-labs$ podman rmi --all
Untagged: docker.io/redhat/ubi8:latest
Untagged: localhost/myapache:0.1
Deleted: ec600407b590f735212f408bf86a7cfd460e6480f0e8319b19620f1b3c8349cd
```