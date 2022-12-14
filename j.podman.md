# Define, build and modify container images

- Note: The topic is part of the new CKAD syllabus. Here are a few examples of using **podman** to manage the life cycle of container images. The use of **docker** had been the industry standard for many years, but now large companies like [Red Hat](https://www.redhat.com/en/blog/say-hello-buildah-podman-and-skopeo) are moving to a new suite of open source tools: podman, skopeo and buildah. Also Kubernetes has moved in this [direction](https://kubernetes.io/blog/2022/02/17/dockershim-faq/). In particular, `podman` is meant to be the replacement of the `docker` command: so it makes sense to get familiar with it, although they are quite interchangeable considering that they use the same syntax.

## Podman basics

### Create a Dockerfile to deploy an Apache HTTP Server which hosts a custom main page

Podman is open source. Docker does the same thing. Both provide a container engine for building and running images. Podman is daemonless. I don't know what a daemon is.

Don't know what is necessary for a Dockerfile, but a basic one is below

FROM imagename [this is the only necessary thing]

RUN

COMMAND

<details><summary>show</summary>
<p>

```Dockerfile
FROM httpd:2.4
RUN echo "Hello, World!" > /usr/local/apache2/htdocs/index.html
```

</p>
</details>

### Build and see how many layers the image consists of

podman build .

used podman image --help to get 
podman image inspect 8fa887cacc2c
Where 8fa etc is the image hash id
ctrl-f for layers showed there were 6

Actual method is podman image tree 8fa887cacc2c

<details><summary>show</summary>
<p>

```bash
:~$ podman build -t simpleapp .
STEP 1/2: FROM httpd:2.4
STEP 2/2: RUN echo "Hello, World!" > /usr/local/apache2/htdocs/index.html
COMMIT simpleapp
--> ef4b14a72d0
Successfully tagged localhost/simpleapp:latest
ef4b14a72d02ae0577eb0632d084c057777725c279e12ccf5b0c6e4ff5fd598b

:~$ podman images
REPOSITORY               TAG         IMAGE ID      CREATED        SIZE
localhost/simpleapp      latest      ef4b14a72d02  8 seconds ago  148 MB
docker.io/library/httpd  2.4         98f93cd0ec3b  7 days ago     148 MB

:~$ podman image tree localhost/simpleapp:latest
Image ID: ef4b14a72d02
Tags:     [localhost/simpleapp:latest]
Size:     147.8MB
Image Layers
????????? ID: ad6562704f37 Size:  83.9MB
????????? ID: c234616e1912 Size: 3.072kB
????????? ID: c23a797b2d04 Size: 2.721MB
????????? ID: ede2e092faf0 Size: 61.11MB
????????? ID: 971c2cdf3872 Size: 3.584kB Top Layer of: [docker.io/library/httpd:2.4]
????????? ID: 61644e82ef1f Size: 6.144kB Top Layer of: [localhost/simpleapp:latest]
```

</p>
</details>

### Run the image locally, inspect its status and logs, finally test that it responds as expected

podman run --name test -d -p 8080:80 8fa887cacc2c 

Then localhost:8080, responds with Hello, World!

<details><summary>show</summary>
<p>

```bash
:~$ podman run -d --name test -p 8080:80 localhost/simpleapp
2f3d7d613ea6ba19703811d30704d4025123c7302ff6fa295affc9bd30e532f8

:~$ podman ps
CONTAINER ID  IMAGE                       COMMAND           CREATED        STATUS            PORTS                 NAMES
2f3d7d613ea6  localhost/simpleapp:latest  httpd-foreground  5 seconds ago  Up 6 seconds ago  0.0.0.0:8080->80/tcp  test

:~$ podman logs test
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.0.2.100. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.0.2.100. Set the 'ServerName' directive globally to suppress this message
[Sat Jun 04 16:15:38.071377 2022] [mpm_event:notice] [pid 1:tid 139756978220352] AH00489: Apache/2.4.53 (Unix) configured -- resuming normal operations
[Sat Jun 04 16:15:38.073570 2022] [core:notice] [pid 1:tid 139756978220352] AH00094: Command line: 'httpd -D FOREGROUND'

:~$ curl 0.0.0.0:8080
Hello, World!
```

</p>
</details>

### Run a command inside the pod to print out the index.html file

podman exec ecbabdc59f88 ls
i.e. put the command immediately after
podman exec ecbabdc59f88 cat /usr/local/apache2/htdocs/index.html
Because in the dockerfile we did:
RUN echo "Hello, World!" > /usr/local/apache2/htdocs/index.html

<details><summary>show</summary>
<p>

```bash
:~$ podman exec -it test cat /usr/local/apache2/htdocs/index.html
Hello, World!
```
</p>
</details>

### Tag the image with ip and port of a private local registry and then push the image to this registry

podman tag IMAGE TAG

No idea how to set up a local image registry
- https://www.techrepublic.com/article/how-to-set-up-a-local-image-repository-with-podman/
- Create a directory
- Initialise it as a podman registry
- Configure, setting local host and port
- Push to local registry using local host, port, and labels

I'm hoping that CKAD won't go into this much detail. It should only require pushing, pulling, labelling, using urls, etc

<details><summary>show</summary>
<p>

> Note: Some small distributions of Kubernetes (such as [microk8s](https://microk8s.io/docs/registry-built-in)) have a built-in registry you can use for this exercise. If this is not your case, you'll have to setup it on your own.

```bash
:~$ podman tag localhost/simpleapp $registry_ip:5000/simpleapp

:~$ podman push $registry_ip:5000/simpleapp
```

</p>
</details>

 Verify that the registry contains the pushed image and that you can pull it

<details><summary>show</summary>
<p>

```bash
:~$ curl http://$registry_ip:5000/v2/_catalog
{"repositories":["simpleapp"]}

# remove the image already present
:~$ podman rmi $registry_ip:5000/simpleapp

:~$ podman pull $registry_ip:5000/simpleapp
Trying to pull 10.152.183.13:5000/simpleapp:latest...
Getting image source signatures
Copying blob 643ea8c2c185 skipped: already exists
Copying blob 972107ece720 skipped: already exists
Copying blob 9857eeea6120 skipped: already exists
Copying blob 93859aa62dbd skipped: already exists
Copying blob 8e47efbf2b7e skipped: already exists
Copying blob 42e0f5a91e40 skipped: already exists
Copying config ef4b14a72d done
Writing manifest to image destination
Storing signatures
ef4b14a72d02ae0577eb0632d084c057777725c279e12ccf5b0c6e4ff5fd598b
```

</p>
</details>

### Run a pod with the image pushed to the registry

Might be good to go back to this

<details><summary>show</summary>
<p>

```bash
:~$ kubectl run simpleapp --image=$registry_ip:5000/simpleapp --port=80

:~$ curl ${kubectl get pods simpleapp -o jsonpath='{.status.podIP}'}
Hello, World!
```

</p>
</details>

### Log into a remote registry server and then read the credentials from the default file



<details><summary>show</summary>
<p>

> Note: The two most used container registry servers with a free plan are [DockerHub](https://hub.docker.com/) and [Quay.io](https://quay.io/).

```bash
:~$ podman login --username $YOUR_USER --password $YOUR_PWD docker.io
:~$ cat ${XDG_RUNTIME_DIR}/containers/auth.json
{
        "auths": {
                "docker.io": {
                        "auth": "Z2l1bGl0JLSGtvbkxCcX1xb617251xh0x3zaUd4QW45Q3JuV3RDOTc="
                }
        }
}
```

</p>
</details>

### Create a secret both from existing login credentials and from the CLI

<details><summary>show</summary>
<p>

```bash
:~$ kubectl create secret generic mysecret --from-file=.dockerconfigjson=${XDG_RUNTIME_DIR}/containers/auth.json --type=kubernetes.io/dockeconfigjson
secret/mysecret created
:~$ kubectl create secret docker-registry mysecret2 --docker-server=https://index.docker.io/v1/ --docker-username=$YOUR_USR --docker-password=$YOUR_PWD
secret/mysecret2 created
```

</p>
</details>

### Create the manifest for a Pod that uses one of the two secrets just created to pull an image hosted on the relative private remote registry

<details><summary>show</summary>
<p>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: $YOUR_PRIVATE_IMAGE
  imagePullSecrets:
  - name: mysecret
```

</p>
</details>

### Clean up all the images and containers

<details><summary>show</summary>
<p>

```bash
:~$ podman rm --all --force
:~$ podman rmi --all
:~$ kubectl delete pod simpleapp
```

</p>
</details>
