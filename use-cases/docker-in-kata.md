# Running Docker within a Kata Container

## Create a custom kernel

### Get the code

```
go get github.com/kata-containers/tests
cd $GOPATH/src/github.com/kata-containers/tests/.ci
```

### Download the standard kata-container kernel config and the linux kernel

```
kernel_arch="$(./kata-arch.sh)"
kernel_dir="$(./kata-arch.sh --kernel)"
tmpdir="$(mktemp -d)"
pushd "$tmpdir"
curl -L https://raw.githubusercontent.com/kata-containers/packaging/master/kernel/configs/${kernel_dir}_kata_kvm_4.14.x -o .config
kernel_version=$(grep "Linux/[${kernel_arch}]*" .config | cut -d' ' -f3 | tail -1)
kernel_tar_file="linux-${kernel_version}.tar.xz"
kernel_url="https://cdn.kernel.org/pub/linux/kernel/v$(echo $kernel_version | cut -f1 -d.).x/${kernel_tar_file}"
curl -LOk ${kernel_url}
tar -xf ${kernel_tar_file}
mv .config "linux-${kernel_version}"
pushd "linux-${kernel_version}"
curl -L https://raw.githubusercontent.com/kata-containers/packaging/master/kernel/patches/0001-NO-UPSTREAM-9P-always-use-cached-inode-to-fill-in-v9.patch | patch -p1
curl -L https://raw.githubusercontent.com/kata-containers/packaging/master/kernel/patches/0002-Compile-in-evged-always.patch | patch -p1
```

### Customize the kernel

Edit the `.config` file


```
CONFIG_OVERLAY_FS=y
CONFIG_OVERLAY_FS_REDIRECT_DIR=y
CONFIG_OVERLAY_FS_INDEX=y
```

Alternatively you could use `make menuconfig` command:

```
follow the menu

'File systems -> Overlay File System Support -> Overlayfs'
```

### Build the kernel

```
make ARCH=${kernel_dir} -j$(nproc)
```

### Replace the default kernel with the custom one

```
kata_kernel_dir="/usr/share/kata-containers"
kata_vmlinuz="${kata_kernel_dir}/kata-vmlinuz-${kernel_version}.container"
[ $kernel_arch = ppc64le ] && kernel_file="$(realpath ./vmlinux)" || kernel_file="$(realpath arch/${kernel_arch}/boot/bzImage)"
install -o root -g root -m 0755 -D "${kernel_file}" "${kata_vmlinuz}"
ln -sf "${kata_vmlinuz}" "${kata_kernel_dir}/vmlinuz.container"
kata_vmlinux="${kata_kernel_dir}/kata-vmlinux-${kernel_version}"
install -o root -g root -m 0755 -D "$(realpath vmlinux)" "${kata_vmlinux}"
ln -sf "${kata_vmlinux}" "${kata_kernel_dir}/vmlinux.container"
popd
popd
```

## Example

**Pod's template** `toolbox.yaml`:

```
apiVersion: v1
kind: Pod
metadata:
  name: toolbox
  annotations:
    io.kubernetes.cri.untrusted-workload: "true"
spec:
  containers:
  - name: toolbox
    image: ubuntu
    args: ["sleep", "3600"]
  - name: dind
    securityContext:
      privileged: true
    image: docker:18.06.1-ce-dind
    args: ["--storage-driver=vfs"]
    volumeMounts:
      - mountPath: /var/run/
        name: dockersock
    args: ["sleep", "3600"]
  volumes:
  - name: dockersock
    emptyDir: {}
```

**Test run**:

```
kubectl apply -f toolbox.yaml
kubectl exec -it toolbox sh -c dind

/ # dockerd &

/ # docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 18.06.1-ce
Storage Driver: overlay2
 Backing Filesystem: extfs
 Supports d_type: true
 Native Overlay Diff: false
```