
# Portworx and Kubernetes
Portworx PX-Lite is elastic block storage for containers. Deploying PX-Lite on a server with Docker turns that server into a scale-out storage node. This quick guide shows how to use PX-Lite to implement storage for Kubernetes pods. 

Future versions will support launching PX-Lite through Kubernetes. For more on PX-Lite, see our [quick start guides](https://github.com/portworx/px-lite#install-and-quick-start-guides). 

## Using PX-Lite with Kubernetes
PX-lite pools your servers capacity and is deployed as a container. Here is how to install PX-Lite on each server. We are tracking when shared mounts will be allowed within Kubernetes (K8s), which will allow Kubernetes to deploy PX-Lite. 

### Step 1: Run the PX-Lite container outside of Kubernetes

Run PX-Lite container using docker with following command

```
$ docker run --restart=always --name px-lite -d --net=host
--privileged=true \
-v /run/docker/plugins:/run/docker/plugins \
-v /var/lib/osd:/var/lib/osd \
-v /dev:/dev \
-v /etc/pwx:/etc/pwx \
-v /opt/pwx/bin:/export_bin \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /var/cores:/var/cores \
-v /var/lib/kubelet:/var/lib/kubelet:shared \
--ipc=host \
portworx/px-lite:latest
```

### Step 2: Install Flexvolume Binary on all Kubernetes nodes
Flexvolume allows other volume drivers outside of Kubernetes to
attach/detach/mount/unmount custom volumes to pods/daemonsets/rcs

You can download the flexvolume binary built for x86_64 arch from [S3](https://s3-us-west-1.amazonaws.com/kubernetes-portworx/flexvolume)

OR

Build the flexvolume binary in openstorage.
```
$ git clone git@github.com:libopenstorage/openstorage.git
$ cd libopenstorage/openstorage
$ make
```

Copy the compiled binary into the Kubernetes plugin path on all the k8s nodes
```
$ mkdir /usr/libexec/kubernetes/kubelet-plugins/volume/exec/px~flexvolume/flexvolume
$ cp ../../../bin/flexvolume /usr/libexec/kubernetes/kubelet-plugins/volume/exec/px~flexvolume/flexvolume
```

### Step 3: Include PX Flexvolume as a VolumeSpec in Kubernetes spec file

Under "spec" section of your spec yaml file, add a "volumes" section.

``` yaml
spec:
  volumes:
    - name: test
      flexVolume:
        driver: "px/flexvolume"
        fsType: "ext4"
        options:
          volumeID: "615055680017358399"
          size: "1G"
          osdDriver: "pxd"
```
* The driver name px/flexvolume should match the plugin directory
(px~flexvolume) which was created in the previous step.
* Specify the unique ID for the volume create in px-lite container as
the volumeID field.
* Set the osdDriver to "pxd" always. It indicates that the flexvolume
should use the px driver for managing volumes.

### Step 4: Include the Flexvolume as a VolumeMount spec in your container/application.

Once you have specified flexvolume as a volume type in your spec
file, you can mount it by including "volumeMounts" spec

Here is an example how you could use it in your container

``` yaml
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
    volumeMounts:
    - name: test
      mountPath: /data
```

* Use the same "name" field as used while defining the volume.

### Step 5: Run the Kubernetes cluster in privileged mode.

* In order to share the namespace between the host, PX-Lite container
  and your Kubernetes pod instance you need to run the cluster with 
  privileges. This can be done by setting the environment variable
  "ALLOW_PRIVILEGED" equal to "true"
* Share the host path "/var/lib/kubelet" with PX-Lite container and
  your pods. The above docker run command for PX-Lite shares this
  path. In order to share it within your pod add a new "hostPath" type
  volume and a corresponding volumeMount in your spec file.

```yaml
spec:
  containers:
    volumeMounts:
    - name: lib
      mountPath: /var/lib/kubelet
  volumes:
    - name: lib
      hostPath:
        path: /var/lib/kubelet

```


## Summary of steps

* Run px-lite container using docker with following command

```
$ docker run --restart=always --name px-lite -d --net=host
--privileged=true \
-v /run/docker/plugins:/run/docker/plugins \
-v /var/lib/osd:/var/lib/osd \
-v /dev:/dev \
-v /etc/pwx:/etc/pwx \
-v /opt/pwx/bin:/export_bin \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /var/cores:/var/cores \
-v /var/lib/kubelet:/var/lib/kubelet:shared \
--ipc=host \
portworx/px-lite:latest
```

* Copy the flexvolume binary to the kubernetes plugin path on all
  nodes

```
$ mkdir /usr/libexec/kubernetes/kubelet-plugins/volume/exec/px~flexvolume/flexvolume
$ cp ../../../bin/flexvolume /usr/libexec/kubernetes/kubelet-plugins/volume/exec/px~flexvolume/flexvolume
```

* Start your k8s cluster in privileged mode

```
$ cd kubernetes
$ ALLOW_PRIVILEGED=true hack/local-up-cluster.sh
```

* Set your cluster details

```
$ cluster/kubectl.sh config set-cluster local --server=http://127.0.0.1:8080 --insecure-skip-tls-verify=true
$ cluster/kubectl.sh config set-context local --cluster=local
$ cluster/kubectl.sh config use-context local
```

* Run your pod

```
$ ./kubectl create -f nginx-pxd.yaml
```
