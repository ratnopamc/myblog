---
layout: post
title:  "Validating Kubernetes Clusters With Sonobuoy"
date:   2021-05-23 01:16:12 -0400
categories: jekyll update
---

# Welcome to Conformance Testing With SOnobuoy

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