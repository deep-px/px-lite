## Using px-lite with k8s

### Step 1: Run the px-lite container outside of k8s

Run px-lite container using docker with following command

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

### Step 2: Install Flexvolume Binary on all k8s nodes
Flexvolume allows other volume drivers outside of k8s to
attach/detach/mount/unmount custom volumes to pods/daemonsets/rcs

Build the flexvolume in openstorage and create a binary. Copy the
compiled binary into the kubernetes plugin path on all the k8s nodes
```
$ cd libopenstorage/openstorage
$ make
$ mkdir /usr/libexec/kubernetes/kubelet-plugins/volume/exec/px~flexvolume/flexvolume
$ cp ../../../bin/flexvolume /usr/libexec/kubernetes/kubelet-plugins/volume/exec/px~flexvolume/flexvolume
```

### Step 3: Include PX Flexvolume as a VolumeSpec in k8s spec file

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

### Step 5: Run the k8s cluster in privileged mode.

* In order to share the namespace between the host, px-lite container
  and your k8s pod instance you need to run the cluster with highest
  privilege. This can be done by setting the environment variable
  "ALLOW_PRIVILEGED" equal to "true"
* Share the host path "/var/lib/kubelet" with px-lite container and
  your pods. The above docker run command for px-lite shares this
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


## tl;dr

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
