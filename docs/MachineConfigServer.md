# MachineConfigServer

## Goals

1. Provide Ignition config to new machines joining the cluster.

## Non Goals

## Overview

All the machines joining the cluster must receive configuration from component running inside the cluster. MachineConfigServer provides the Ignition endpoint that all the new machines can point to receive their machine configuration. 

The machine can request specific configuration by pointing Ignition to MachineConfigServer and passing appropriate parameters. For example, to fetch configuration for master the machine can point to `/config/master` whereas, to fetch configuration for worker the machine can point to `/config/worker`.

## Detailed Design

### Endpoint

MachineConfigServer serves Ignition at `/config/<machine-pool-name>` endpoint.

* If the server finds the machine pool requested in the URL, it returns the Ignition config stored in the MachineConfig object referenced at `.status.currentMachineConfig` in the MachinePool object.

* If the server cannot find the machine pool requested in the URL, the server returns HTTP Status Code 404 with an empty response.

### Special parameter for master machine configuration

The etcd members are co-located with the control plane on `master` machines. Bootstrapping an etcd cluster requires special code path that depends on the index of the machine in the pool. Therefore, the server needs to support `etcd_index` query parameter for `master` machine pools to serve the correct Ignition config.

### Ignition config from MachineConfig

MachineConfigServer serves the Ignition config defined in `spec.config` fields of the appropriate MachineConfig object.

It performs the following extra actions on the Ignition config defined in the MachineConfig object before serving it:

* *etcd member index templating*

    The etcd unit files generated by MachineConfigController are go text templates with `etcd_index` variable. This variable needs to replaces with the index specified in `etcd_index` query.

* *Ignition file for MachineConfigDaemon*

    MachineConfigDaemon requires a file on disk (node annotations), to seed the `currentConfig` & `desiredConfig` annotations to its node object. The file is JSON object that contains the reference to `MachineConfig` object used to generate the Ignition config for the machine.

* *Ignition file for KubeConfig*

   The new machines that come up, will need a KubeConfig file which will be added as an Ignition file. 

### Running MachineConfigServer

It is recommended that the MachineConfigServer is run as a DaemonSet on all `master` machines with the pods running in host network. So machines can access the Ignition endpoint through load balancer setup for control plane.

### Example requests

1. Worker machine

    Request:

    GET `/config/worker`

    Response:

    **TODO(abhinavdahiya): add example response**

2. Master machine with etcd member `etcd-1`

    Request:

    GET `/config/master?etcd_index=1`

    Response:

```json
   {
	"ignition": {
		"config": {},
		"security": {
			"tls": {}
		},
		"timeouts": {}
	},
	"networkd": {},
	"passwd": {},
	"storage": {
		"files": [{
			"contents": {
				"verification": {}
			}
		}, {
			"filesystem": "root",
			"path": "/etc/machine-config-daemon/node-annotations.json",
			"contents": {
				"source": "data:,%7B%22machineconfiguration.openshift.io%2FcurrentConfig%22%3A%22test-config%22%2C%22machineconfiguration.openshift.io%2FdesiredConfig%22%3A%22test-config%22%7D",
				"verification": {}
			},
			"mode": 420
		}, {
			"filesystem": "root",
			"path": "/etc/system/kubeconfig",
			"contents": {
				"source": "data:,apiVersion%3A%20v1%0Aclusters%3A%0A-%20cluster%3A%0A%20%20%20%20certificate-authority-data%3A%20LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURkVENDQWwyZ0F3SUJBZ0lSQUs1aGNubDBBMGhrcnArU2tHdTg2ak13RFFZSktvWklodmNOQVFFTEJRQXcKVkRFdE1Dc0dBMVVFQ2hNa09UZzJaalZsWldJdE56WXpaUzB5WVRSbUxURTRaRFV0TTJWa01HRmxaRGszWkdRegpNUkV3RHdZRFZRUUxFd2gwWldOMGIyNXBZekVRTUE0R0ExVUVBeE1IEs5SUxiWGgvRGpzQUpFcWJCVnNIMTQ4SStwOWc9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg%3D%3D%0A%20%20%20%20server%3A%20https%3A%2F%2Ftest-system%3A443%0A%20%20name%3A%20mcs%0Acontexts%3A%0A-%20context%3A%0A%20%20%20%20cluster%3A%20mcs%0A%20%20%20%20user%3A%20admin%0A%20%20name%3A%20%22%22%0A-%20context%3A%0A%20%20%20%20cluster%3A%20mcs%0A%20%20%20%20namespace%3A%20kube-system%0A%20%20%20%20user%3A%20admin%0A%20%20name%3A%20kube-system%0Acurrent-context%3A%20kube-system%0Akind%3A%20Config%0Apreferences%3A%20%7B%7D%0Ausers%3A%0A-%20name%3A%20admin%0A%20%20user%3A%0A%20%20%20%20client-certificate-data%3A%20LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURhRENDQWxDZ0F3SUJBZ0lSQVA2RnYxK2RlSE9JNFRUTm5RNk5sUFl3RFFZSktvWklodmNOQVFFTEJRQXcKVkRFdE1Dc0dBMVVFQ2hNa05qRTBZemM1TldFdE9UTXpaQzB3WkRjM0xURTVOell0Tm1ReVpUazFNRFpoWTJSawpNUkV3RHdZRFZRUUxFd2hpYjI5MGEzVmlaVEVRTUE0R0ExVUVBeE1IYTNWaVpZUtpRmM4bzRMVVh1V1FhMUtleW5VCnpwWGROOUxpK0JJYnFDb0ZoZVpiTk5YM3hUSm5LNWFTOVVtWTliYXlERmhWOUl1RHVpWTBPWHB0TWxHQ243ZkIKaGh0YjFsUTcwaHV4Wno3OEJnSjZMU2UrcEt2eFBuaVpQZ1hFNlhCdmdkSmRjZU5ZZmJlUTFrbHoxRG1JSmIxYgpGSExNZVBxY2txMEkyaFJZYXJ5UDBlbVJjbUJqSDZpaWl1UnRBeTFUY1ZnTTBTZzhZbjFFSExCbnhiUm41TmpsCnZSU3c1MnRSUUtBbE5WeDEKLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo%3D%0A%20%20%20%20client-key-data%3A%20RWGdnT0huTmZYZDlxUnQ5TmdyVE5rMGh2Tzk0eApRYkxJUHNjM0dDTU1FQm8zZnlUait3ZkJTeFBydTg0Q3BRYjdxS3A2Q3FqcDNGSDBuT2l2ejNIU3JrRkF2cU5WClVpd001Q3ZFQWt0dTI5NU0zUE42YlNrQ2dZQTNLSi9pYUFqVW1IK3V2SjFXUFpMTnBtaDdEdmdxcmd6aEFoSHEKZ2V1NkpTNXpRYktyNkYvYmVmOVgxd1BLdmcxbDZ3WmVMc3cvZkNHMUY0c3U0NlVVY3ZDVlVRMm94eE40anc3SwozUWhiaWpxS3N5dmhPYzBtQ1FlemgzblRsOUtaVjJlOTllWUVvc0NIRU1ZekpMN3BiWW02SFpjajI5QjRsY21tCjM1bGhLd0tCZ1FDYitUQ2x6ZVFhakdWSWI0UXBtcjJ3R2xFamEzUytFTFNwUEVJZmNxUUdvWjcrMWdxMHRhOWsKT2Zvb0p2Y1NpVWZoSnJTbHJUSk1NTmFDT0ZUR0tZNVlKcGMrMzlndXY1YjI0aHVGZmdGeWJNT09aek1VekVWSwo2UHM1Z2hBQjh3c05oZTZHOHRBbytIZUduVU9DdE94YmlrMWp6RjkvQmJFaXE3Lzc2SDMyd0E9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo%3D%0A",
				"verification": {}
			},
			"mode": 420
		}]
	},
	"systemd": {
		"units": [{
			"dropins": [{
				"contents": "[Service]\nEnvironment=\"DOCKER_OPTS=--log-opt max-size=50m --log-opt max-file=3\n",
				"name": "10-dockeropts.conf"
			}],
			"enabled": true,
			"name": "docker.service"
		}, {
			"contents": "[Unit]\nDescription=etcd (System Application Container) TLS assets\nConditionFileNotEmpty=|!/etc/ssl/etcd/system:etcd-server:my-test-cluster-etcd-1.installer.team.coreos.systems.crt\nConditionFileNotEmpty=|!/etc/ssl/etcd/system:etcd-server:my-test-cluster-etcd-1.installer.team.coreos.systems.key\nConditionFileNotEmpty=|!/etc/ssl/etcd/system:etcd-peer:my-test-cluster-etcd-1.installer.team.coreos.systems.crt\nConditionFileNotEmpty=|!/etc/ssl/etcd/system:etcd-peer:my-test-cluster-etcd-1.installer.team.coreos.systems.key\nRequires=docker.service\nAfter=docker.service\n\n[Service]\nType=oneshot\nRemainAfterExit=yes\n\nEnvironment=\"SIGNER_IMAGE=quay.io/coreos/kube-client-agent:678cc8e6841e2121ebfdb6e2db568fce290b67d6\"\n\nExecStart=/usr/bin/docker \\\n  run \\\n    --rm \\\n    --volume /etc/ssl/etcd:/etc/ssl/etcd:rw \\\n  \"${SIGNER_IMAGE}\" \\\n    request \\\n      --orgname=system:etcd-servers \\\n      --cacrt=/etc/ssl/etcd/root-ca.crt \\\n      --assetsdir=/etc/ssl/etcd \\\n      --address=https://my-test-cluster-api.installer.team.coreos.systems:6443 \\\n      --dnsnames=localhost,*.kube-etcd.kube-system.svc.cluster.local,kube-etcd-client.kube-system.svc.cluster.local,my-test-cluster-etcd-0.installer.team.coreos.systems,my-test-cluster-etcd-1.installer.team.coreos.systems,my-test-cluster-etcd-2.installer.team.coreos.systems \\\n      --commonname=system:etcd-server:my-test-cluster-etcd-1.installer.team.coreos.systems \\\n      --ipaddrs=127.0.0.1 \\\n\nExecStart=/usr/bin/docker \\\n  run \\\n    --rm \\\n    --volume /etc/ssl/etcd:/etc/ssl/etcd:rw \\\n  \"${SIGNER_IMAGE}\" \\\n    request \\\n      --orgname=system:etcd-peers \\\n      --cacrt=/etc/ssl/etcd/root-ca.crt \\\n      --assetsdir=/etc/ssl/etcd \\\n      --address=https://my-test-cluster-api.installer.team.coreos.systems:6443 \\\n      --dnsnames=*.kube-etcd.kube-system.svc.cluster.local,kube-etcd-client.kube-system.svc.cluster.local,my-test-cluster-etcd-0.installer.team.coreos.systems,my-test-cluster-etcd-1.installer.team.coreos.systems,my-test-cluster-etcd-2.installer.team.coreos.systems \\\n      --commonname=system:etcd-peer:my-test-cluster-etcd-1.installer.team.coreos.systems \\\n\nExecStart=/bin/chown etcd:etcd /etc/ssl/etcd/system:etcd-server:my-test-cluster-etcd-1.installer.team.coreos.systems.crt\nExecStart=/bin/chown etcd:etcd /etc/ssl/etcd/system:etcd-server:my-test-cluster-etcd-1.installer.team.coreos.systems.key\nExecStart=/bin/chown etcd:etcd /etc/ssl/etcd/system:etcd-peer:my-test-cluster-etcd-1.installer.team.coreos.systems.crt\nExecStart=/bin/chown etcd:etcd /etc/ssl/etcd/system:etcd-peer:my-test-cluster-etcd-1.installer.team.coreos.systems.key\n",
			"enabled": true,
			"name": "etcd-member-tls.service"
		}, {
			"contents": "[Unit]\nDescription=etcd (System Application Container)\nDocumentation=https://github.com/coreos/etcd\nRequires=etcd-member-tls.service\nAfter=etcd-member-tls.service\n\n[Service]\nRestart=on-failure\nRestartSec=10s\nTimeoutStartSec=0\nLimitNOFILE=40000\n\nEnvironment=\"ETCD_IMAGE=quay.io/coreos/etcd:v3.2.14\"\n\nExecStartPre=-/usr/bin/docker rm etcd-member\nExecStartPre=/usr/bin/mkdir --parents /var/lib/etcd\nExecStartPre=/usr/bin/mkdir --parents /run/etcd\nExecStartPre=/usr/bin/chown etcd /var/lib/etcd\nExecStartPre=/usr/bin/chown etcd /run/etcd\n\nExecStart= /usr/bin/bash -c \" \\\n    /usr/bin/docker \\\n      run \\\n        --rm \\\n        --name etcd-member \\\n        --volume /run/systemd/system:/run/systemd/system:ro \\\n        --volume /etc/ssl/certs:/etc/ssl/certs:ro \\\n        --volume /etc/ssl/etcd:/etc/ssl/etcd:ro \\\n        --volume /var/lib/etcd:/var/lib/etcd:rw \\\n        --volume /etc/ssl/certs:/etc/ssl/certs:ro \\\n        --env 'ETCD_NAME=%m' \\\n        --env ETCD_DATA_DIR=/var/lib/etcd \\\n        --network host \\\n        --user=$(id --user etcd) \\\n      '${ETCD_IMAGE}' \\\n        /usr/local/bin/etcd \\\n          --name=my-test-cluster-etcd-1.installer.team.coreos.systems \\\n          --advertise-client-urls=https://my-test-cluster-etcd-1.installer.team.coreos.systems:2379 \\\n          --cert-file=/etc/ssl/etcd/system:etcd-server:my-test-cluster-etcd-1.installer.team.coreos.systems.crt \\\n          --key-file=/etc/ssl/etcd/system:etcd-server:my-test-cluster-etcd-1.installer.team.coreos.systems.key \\\n          --trusted-ca-file=/etc/ssl/etcd/ca.crt \\\n          --client-cert-auth=true \\\n          --peer-cert-file=/etc/ssl/etcd/system:etcd-peer:my-test-cluster-etcd-1.installer.team.coreos.systems.crt \\\n          --peer-key-file=/etc/ssl/etcd/system:etcd-peer:my-test-cluster-etcd-1.installer.team.coreos.systems.key \\\n          --peer-trusted-ca-file=/etc/ssl/etcd/ca.crt \\\n          --peer-client-cert-auth=true \\\n          --initial-cluster='my-test-cluster-etcd-0.installer.team.coreos.systems=https://my-test-cluster-etcd-0.installer.team.coreos.systems:2380,my-test-cluster-etcd-1.installer.team.coreos.systems=https://my-test-cluster-etcd-1.installer.team.coreos.systems:2380,my-test-cluster-etcd-2.installer.team.coreos.systems=https://my-test-cluster-etcd-2.installer.team.coreos.systems:2380' \\\n          --initial-advertise-peer-urls=https://my-test-cluster-etcd-1.installer.team.coreos.systems:2380 \\\n          --listen-client-urls=https://0.0.0.0:2379 \\\n          --listen-peer-urls=https://0.0.0.0:2380 \\\n    \"\n\n[Install]\nWantedBy=multi-user.target\n",
			"enabled": true,
			"name": "etcd-member.service"
		}, {
			"contents": "[Service]\nExecStart=/usr/bin/env bash -c \\\n  \" \\\n    if grep rhcos /etc/os-release \u003e /dev/null; \\\n    then \\\n      echo CGROUP_DRIVER_FLAG=--cgroup-driver=systemd \u003e /etc/kubernetes/kubelet-workaround; \\\n      mount -o remount,rw /sys/fs/cgroup; \\\n      ln --symbolic /sys/fs/cgroup/cpu,cpuacct /sys/fs/cgroup/cpuacct,cpu; \\\n    fi \\\n  \"\n",
			"name": "kubelet-workaround.service"
		}, {
			"contents": "[Unit]\nDescription=Kubernetes Kubelet\nWants=rpc-statd.service\nRequires=docker.service kubelet-workaround.service\nAfter=docker.service kubelet-workaround.service\n\n[Service]\nEnvironment=\"KUBELET_IMAGE=openshift/origin-node\"\nEnvironmentFile=-/etc/kubernetes/kubelet-workaround\n\nExecStartPre=/bin/mkdir --parents /etc/kubernetes/manifests\nExecStartPre=/bin/mkdir --parents /etc/kubernetes/checkpoint-secrets\nExecStartPre=/bin/mkdir --parents /etc/kubernetes/cni/net.d\nExecStartPre=/bin/mkdir --parents /run/kubelet\nExecStartPre=/bin/mkdir --parents /var/lib/cni\nExecStartPre=/bin/mkdir --parents /var/lib/kubelet/pki\n\nExecStartPre=/usr/bin/bash -c \"gawk '/certificate-authority-data/ {print $2}' /etc/kubernetes/kubeconfig | base64 --decode \u003e /etc/kubernetes/ca.crt\"\n\nExecStart=/usr/bin/docker \\\n  run \\\n    --rm \\\n    --net host \\\n    --pid host \\\n    --privileged \\\n    --volume /dev:/dev:rw \\\n    --volume /sys:/sys:ro \\\n    --volume /var/run:/var/run:rw \\\n    --volume /var/lib/cni/:/var/lib/cni:rw \\\n    --volume /var/lib/docker/:/var/lib/docker:rw \\\n    --volume /var/lib/kubelet/:/var/lib/kubelet:shared \\\n    --volume /var/log:/var/log:shared \\\n    --volume /etc/kubernetes:/etc/kubernetes:ro \\\n    --entrypoint /usr/bin/hyperkube \\\n  \"${KUBELET_IMAGE}\" \\\n    kubelet \\\n      --bootstrap-kubeconfig=/etc/kubernetes/kubeconfig \\\n      --kubeconfig=/var/lib/kubelet/kubeconfig \\\n      --rotate-certificates \\\n      --cni-conf-dir=/etc/kubernetes/cni/net.d \\\n      --cni-bin-dir=/var/lib/cni/bin \\\n      --network-plugin=cni \\\n      --lock-file=/var/run/lock/kubelet.lock \\\n      --exit-on-lock-contention \\\n      --pod-manifest-path=/etc/kubernetes/manifests \\\n      --allow-privileged \\\n      --node-labels=node-role.kubernetes.io/etcd \\\n      --minimum-container-ttl-duration=6m0s \\\n      --cluster-dns=10.3.0.10 \\\n      --cluster-domain=cluster.local \\\n      --client-ca-file=/etc/kubernetes/ca.crt \\\n      --cloud-provider=aws \\\n       \\\n      --anonymous-auth=false \\\n      --register-with-taints=node-role.kubernetes.io/etcd=:NoSchedule \\\n      $CGROUP_DRIVER_FLAG \\\n\nRestart=always\nRestartSec=10\n\n[Install]\nWantedBy=multi-user.target\n",
			"enabled": true,
			"name": "kubelet.service"
		}, {
			"mask": true,
			"name": "locksmith.service"
		}]
	}
} 
```
