# pod and container lifecycle benchmark for kata

This repocitory is mainly focused on the pod and container's lifecircle benchmark for kata. The way to run a kata container is
used cri + containerd + containerd-shim-kata-v2. 

## Install

### Install kata components
```sh
ARCH=$(arch)
sudo sh -c "echo 'deb http://download.opensuse.org/repositories/home:/katacontainers:/releases:/${ARCH}:/master/xUbuntu_$(lsb_release -rs)/ /' > /etc/apt/sources.list.d/kata-containers.list"
curl -sL  http://download.opensuse.org/repositories/home:/katacontainers:/releases:/${ARCH}:/master/xUbuntu_$(lsb_release -rs)/Release.key | sudo apt-key add -
sudo -E apt-get update
sudo -E apt-get -y install kata-runtime kata-proxy kata-shim
```
### Install kata shimv2
```sh
go get github.com/kata-containers/runtime
pushd $GOPATH/src/github.com/kata-containers/runtime
git remote add hyper https://github.com/hyperhq/kata-runtime
git fetch hyper
git checkout -b benchmark hyper/benchmark
make 
sudo -E PATH=$PATH make install
popd

sed -i 's/image =/#image =/' /usr/share/defaults/kata-containers/configuration.toml
```	
### Install critest

```sh
go get https://github.com/lifupan/cri-tools.git
pushd $GOPATH/src/github.com/lifupan/cri-tools
git checkout kata_benchmark
make;make install
```

### Install containerd
```sh
sudo apt-get update
sudo apt-get install libseccomp-dev btrfs-tools git -y
go get github.com/containerd/containerd
pushd $GOPATH/src/github.com/containerd/containerd
make
sudo -E PATH=$PATH make install
popd
```

### Install cni plugin
```sh
go get github.com/containernetworking/plugins
pushd $GOPATH/src/github.com/containernetworking/plugins
./build_linux.sh
mkdir /opt/cni
cp -r bin /opt/cni/
popd
```

## Setup cni plugin
```sh
sudo mkdir -p /etc/cni/net.d
cat >/etc/cni/net.d/10-mynet.conf <<EOF
{
	"cniVersion": "0.2.0",
	"name": "mynet",
	"type": "bridge",
	"bridge": "cni0",
	"isGateway": true,
	"ipMasq": true,
	"ipam": {
		"type": "host-local",
		"subnet": "172.19.0.0/24",
		"routes": [
			{ "dst": "0.0.0.0/0" }
		]
	}
}
EOF
```

## Setup containerd 
### create containerd configure file
```sh
sudo mkdir -p /etc/containerd/
cat > /etc/containerd/config.toml <<EOF
[plugins]
  [plugins.cri]
    sandbox_image = "mirrorgooglecontainers/pause-amd64:3.1"
    [plugins.cri.containerd]
      [plugins.cri.containerd.default_runtime]
	runtime_type = "io.containerd.kata.v2"
[plugins.cri.cni]
    # conf_dir is the directory in which the admin places a CNI conf.
    conf_dir = "/etc/cni/net.d"
EOF
```

### create containerd systemd service
```sh
sudo cat > /etc/systemd/system/containerd.service <<EOF
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd --config /etc/containerd/config.toml
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```
### start containerd
```sh
systemctl enable containerd
systemctl start containerd
```

## Run benchmark test
```sh
critest -benchmark --runtime-endpoint /run/containerd/containerd.sock
```
Then you will get some benchmark outputs as below on the screen
```sh
• [MEASUREMENT]
[k8s.io] PodSandbox
/root/gopath/src/github.com/kubernetes-sigs/cri-tools/pkg/framework/framework.go:72
  benchmark about operations on PodSandbox
  /root/gopath/src/github.com/kubernetes-sigs/cri-tools/pkg/benchmark/pod.go:41
    benchmark about lifecycle of PodSandbox
    /root/gopath/src/github.com/kubernetes-sigs/cri-tools/pkg/benchmark/pod.go:42

    Ran 20 samples:
    create PodSandbox:
      Fastest Time: 2.234s
      Slowest Time: 2.625s
      Average Time: 2.499s ± 0.083s
    PodSandbox status:
      Fastest Time: 0.000s
      Slowest Time: 0.002s
      Average Time: 0.001s ± 0.000s
    stop PodSandbox:
      Fastest Time: 0.287s
      Slowest Time: 0.355s
      Average Time: 0.321s ± 0.019s
    remove PodSandbox:
      Fastest Time: 0.008s
      Slowest Time: 0.015s
      Average Time: 0.009s ± 0.002s
      
• [MEASUREMENT]
[k8s.io] Container
/root/gopath/src/github.com/kubernetes-sigs/cri-tools/pkg/framework/framework.go:72
  benchmark about operations on Container
  /root/gopath/src/github.com/kubernetes-sigs/cri-tools/pkg/benchmark/container.go:39
    benchmark about basic operations on Container
    /root/gopath/src/github.com/kubernetes-sigs/cri-tools/pkg/benchmark/container.go:54

    Ran 20 samples:
    create Container:
      Fastest Time: 0.036s
      Slowest Time: 0.070s
      Average Time: 0.053s ± 0.010s
    start Container:
      Fastest Time: 0.144s
      Slowest Time: 0.222s
      Average Time: 0.165s ± 0.016s
    Container status:
      Fastest Time: 0.001s
      Slowest Time: 0.005s
      Average Time: 0.001s ± 0.001s
    stop Container:
      Fastest Time: 0.149s
      Slowest Time: 0.220s
      Average Time: 0.185s ± 0.019s
    remove Container:
      Fastest Time: 0.007s
      Slowest Time: 0.014s
      Average Time: 0.008s ± 0.001s

• [MEASUREMENT]
[k8s.io] PodSandbox
/root/gopath/src/github.com/kubernetes-sigs/cri-tools/pkg/framework/framework.go:72
  benchmark about start a container from scratch
  /root/gopath/src/github.com/kubernetes-sigs/cri-tools/pkg/benchmark/pod_container.go:47
    benchmark about start a container from scratch
    /root/gopath/src/github.com/kubernetes-sigs/cri-tools/pkg/benchmark/pod_container.go:48

    Ran 20 samples:
    create PodSandbox and container:
      Fastest Time: 2.695s
      Slowest Time: 2.963s
      Average Time: 2.841s ± 0.067s

```
