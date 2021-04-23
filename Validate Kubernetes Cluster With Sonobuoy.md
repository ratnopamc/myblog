# Validate k8s cluster with sonobuoy(KubeWeekly) 

## Create a Kind cluster with multiple nodes

```
~/conformance_test$ cat kind_config.yaml

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
# One control plane node and three "workers".
#
# While these will not add more real compute capacity and
# have limited isolation, this can be useful for testing
# rolling updates etc.
#
# The API-server and other control plane components will be
# on the control-plane node.
#
# You probably don't need this unless you are testing Kubernetes itself.
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```



kind create cluster --name conformance --config conformance_test/kind_config.yaml

kubectl get po -A

$ ll ~/conformance_test
total 20
drwxrwxr-x  2 stack stack 4096 Apr 22 22:32 ./
drwxr-xr-x 12 stack stack 4096 Apr 23 03:44 ../
-rwxrwxr-x  1 stack stack 1181 Apr 21 19:09 install_sonobuoy.sh*
-rw-rw-r--  1 stack stack  475 Apr 21 16:34 kind_config.yaml
-rwxrwxr-x  1 stack stack 2002 Apr 22 22:32 run_conformance_test.sh*

$ cat install_sonobuoy.sh
```bash=
#!/usr/bin/env bash

# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

set -xe

: ${SONOBUOY_VERSION:="0.19.0"}
: ${KUBECONFIG:="$HOME/.kube/config"}
URL="https://github.com/vmware-tanzu/sonobuoy/releases/download/v${SONOBUOY_VERSION}/sonobuoy_${SONOBUOY_VERSION}_linux_amd64.tar.gz"
rm -rf /tmp/sonobuoy
mkdir /tmp/sonobuoy
sudo -E curl -sSLo "/tmp/sonobuoy/sonobuoy_${SONOBUOY_VERSION}_linux_amd64.tar.gz" ${URL}
tar xvf /tmp/sonobuoy/sonobuoy_${SONOBUOY_VERSION}_linux_amd64.tar.gz -C /tmp/sonobuoy/
sudo install -m 755 -o root /tmp/sonobuoy/sonobuoy /usr/local/bin
echo ${KUBECONFIG}
echo "**********  sonobuoy version info ************"
sonobuoy version --kubeconfig ${KUBECONFIG}
```



$ cat run_conformance.sh

```bash=
set -xe
: ${SONOBUOY_VERSION:="0.19.0"}
: ${KUBECONFIG:="$HOME/.kube/config"}
# Available Modes: quick, certified-conformance, non-disruptive-conformance.
# (default quick)
: ${CONFORMANCE_MODE:="quick"}
: ${KUBE_CONFORMANCE_IMAGE_VERSION:="v1.19.0"}
: ${TIMEOUT:=10800}
#: ${TARGET_CLUSTER_CONTEXT:="target-cluster"}
: ${E2E_SKIP:=""}

mkdir -p /tmp/sonobuoy_snapshots/e2e
cd /tmp/sonobuoy_snapshots/e2e

#sonobuoy run --plugin e2e --plugin systemd-logs -m ${CONFORMANCE_MODE} --e2e-skip "${E2E_SKIP}" \
#--kube-conformance-image gcr.io/google-containers/conformance:${KUBE_CONFORMANCE_IMAGE_VERSION} \
#--kubeconfig ${KUBECONFIG} \
#--wait --timeout ${TIMEOUT} \
#--log_dir /tmp/sonobuoy_snapshots/e2e

sonobuoy run --plugin e2e --plugin systemd-logs -m ${CONFORMANCE_MODE} --e2e-skip "${E2E_SKIP}" \
--sonobuoy-image projects.registry.vmware.com/sonobuoy/sonobuoy:${SONOBUOY_VERSION \
--kubeconfig ${KUBECONFIG} \
--wait --timeout ${TIMEOUT} \
--log_dir /tmp/sonobuoy_snapshots/e2e

# Get information on pods
kubectl get all -n sonobuoy --kubeconfig ${KUBECONFIG}

# Check sonobuoy status
sonobuoy status --kubeconfig ${KUBECONFIG}

# Get logs
sonobuoy logs --kubeconfig ${KUBECONFIG}

# Store Results
results=$(sonobuoy retrieve --kubeconfig ${KUBECONFIG})
echo "Results: ${results}"

# Display Results
sonobuoy results $results
ls -ltr /tmp/sonobuoy_snapshots/e2e

# Delete sonobuoy objects
echo "Deleting sonobuoy objects"
sonobuoy delete --wait --kubeconfig ${KUBECONFIG}
```

## Commands


$ kubectl get pods -n sonobuoy
No resources found in sonobuoy namespace.


$ sonobuoy version --kubeconfig ~/.kube/config
Sonobuoy Version: v0.19.0
MinimumKubeVersion: 1.17.0
MaximumKubeVersion: 1.19.99
GitSHA: e03f9ee353717ccc5f58c902633553e34b2fe46a
API Version:  v1.19.1


$ kind get clusters
conformance


```
$ kubectl get po -A
NAMESPACE            NAME                                                READY   STATUS    RESTARTS   AGE
kube-system          coredns-f9fd979d6-qsj59                             1/1     Running   0          35h
kube-system          coredns-f9fd979d6-td8b7                             1/1     Running   0          35h
kube-system          etcd-conformance-control-plane                      1/1     Running   0          35h
kube-system          kindnet-2cb25                                       1/1     Running   0          35h
kube-system          kindnet-f769n                                       1/1     Running   0          10h
kube-system          kindnet-p6jbz                                       1/1     Running   0          35h
kube-system          kindnet-pjfrf                                       1/1     Running   0          35h
kube-system          kube-apiserver-conformance-control-plane            1/1     Running   0          35h
kube-system          kube-controller-manager-conformance-control-plane   1/1     Running   0          35h
kube-system          kube-proxy-85vdl                                    1/1     Running   0          35h
kube-system          kube-proxy-gwg4k                                    1/1     Running   0          35h
kube-system          kube-proxy-tvcnt                                    1/1     Running   0          35h
kube-system          kube-proxy-xzxbm                                    1/1     Running   0          35h
kube-system          kube-scheduler-conformance-control-plane            1/1     Running   0          35h
local-path-storage   local-path-provisioner-78776bfc44-g29zn             1/1     Running   0          35h
```

- Non-disruptive mode

```
$ CONFORMANCE_MODE=non-disruptive-conformance ./conformance_test/run_conformance_test.sh
+ : /home/stack/.kube/config
+ : non-disruptive-conformance
+ : v1.19.0
+ : 10800
+ :
+ mkdir -p /tmp/sonobuoy_snapshots/e2e
+ cd /tmp/sonobuoy_snapshots/e2e
+ sonobuoy run --plugin e2e --plugin systemd-logs -m non-disruptive-conformance --e2e-skip '' --sonobuoy-image projects.registry.vmware.com/sonobuoy/sonobuoy:v0.19.0 --kubeconfig /home/stack/.kube/config --wait --timeout 10800 --log_dir /tmp/sonobuoy_snapshots/e2e
INFO[0000] created object                                name=sonobuoy namespace= resource=namespaces
INFO[0000] created object                                name=sonobuoy-serviceaccount namespace=sonobuoy resource=serviceaccounts
INFO[0000] created object                                name=sonobuoy-serviceaccount-sonobuoy namespace= resource=clusterrolebindings
INFO[0000] created object                                name=sonobuoy-serviceaccount-sonobuoy namespace= resource=clusterroles
INFO[0000] created object                                name=sonobuoy-config-cm namespace=sonobuoy resource=configmaps
INFO[0000] created object                                name=sonobuoy-plugins-cm namespace=sonobuoy resource=configmaps
INFO[0000] created object                                name=sonobuoy namespace=sonobuoy resource=pods
INFO[0000] created object                                name=sonobuoy-aggregator namespace=sonobuoy resource=services
```

```
$ kubectl get pods -n sonobuoy
NAME                                                      READY   STATUS    RESTARTS   AGE
sonobuoy                                                  1/1     Running   0          9s
sonobuoy-e2e-job-2e5104ce27504c25                         2/2     Running   0          7s
sonobuoy-systemd-logs-daemon-set-d722528ac01d41da-8p6c7   2/2     Running   0          7s
sonobuoy-systemd-logs-daemon-set-d722528ac01d41da-ccwch   2/2     Running   0          7s
sonobuoy-systemd-logs-daemon-set-d722528ac01d41da-pmwgw   2/2     Running   0          7s
sonobuoy-systemd-logs-daemon-set-d722528ac01d41da-q9ql5   2/2     Running   0          7s
```

- certified-conformance


- quick


sonobuoy delete


docker hub rate limit issue

