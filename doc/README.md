# Table of Contents

- [Introduction](#introduction)
- [Existing work](#existing-work)
- [Implementation](#implementation)
  * [Supported benchmarks](#supported-benchmarks)
  * [PerfKit Benchmarker fork changes](#perfkit-benchmarker-fork-changes)
    + [Including node IDs in benchmark results](#including-node-ids-in-benchmark-results)
  * [Running benchmarks periodically](#running-benchmarks-periodically)
    + [Docker images](#docker-images)
    + [CronJob files and their structure](#cronjob-files-and-their-structure)
      - [The base CronJob file running an user-defined benchmarks list](#the-base-cronjob-file-running-an-user-defined-benchmarks-list)
        * [Repo cloner InitContainer](#repo-cloner-initcontainer)
      - [Dedicated CronJob files for benchmarks](#dedicated-cronjob-files-for-benchmarks)
  * [Permissions](#permissions)
- [User guide](#user-guide)
  * [Configuration](#configuration)
    + [Number of Kubernetes pods](#number-of-kubernetes-pods)
    + [CronJob frequency](#-cronjob-frequency)
    + [Benchmarks list and Pushgateway address](#benchmarks-list-and-pushgateway-address)
  * [Creating a ServiceAccount](#creating-a-serviceaccount)
  * [Launching benchmarks](#launching-benchmarks)
- [Developer guide](#developer-guide)
- [Conclusions](#conclusions)

# Introduction
Running benchmarks on the cloud is not only useful to compare different providers but also to measure the differences when changing the  operating conditions of the cluster (e.g., updating the underlying physical hardware, replacing the CNI provider, adding more nodes, running different workloads in background, etc.).

[`kubemarks`](https://github.com/marcomicera/kubemarks) focuses on the latter aspect, specifically on [Kubernetes](https://kubernetes.io/): it can run a large [set of benchmarks](https://github.com/marcomicera/kubemarks#supported-benchmarks) and expose their results to the [Prometheus](https://prometheus.io/) monitoring system.

# Existing work
Adapting existing benchmarks to run on [Kubernetes](https://kubernetes.io/) may not be straight-forward, especially when dealing with distributed ones that, by definition, need to involve multiple pods.
Yet there are [some attempts online](https://github.com/jberkus/pgKubernetesTutorial) that try to do so.
On the other hand, retrieving benchmark results from different pods requires way more work than just adapting them to [Kubernetes](https://kubernetes.io/).

This is where [PerfKit Benchmarker](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker) comes into play, an open-source tool by [Google Cloud Platform](https://cloud.google.com/) that contains a set of benchmarks that are ready to be run on several cloud offerings, including [Kubernetes](https://kubernetes.io/).
In short, [PerfKit Benchmarker](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker):
- creates and configures as many [Kubernetes](https://kubernetes.io/) pods as needed by the benchmark,
- handles their lifecycle,
- installs dependencies,
- runs benchmarks,
- retrieves results from all pods and, lastly,
- makes it easy to add additional "results writers" so that results can be exported in different ways.

# Implementation
This section aims to describe some system and implementation details needed to accomplish what has been described in the [introduction](#introduction).

## Supported benchmarks
There are four main categories of [PerfKit Benchmarker](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker)-supported benchmarks runnable on [Kubernetes](https://kubernetes.io/):
- I/O-based (e.g., [fio](https://github.com/axboe/fio)),
- networking-oriented (e.g., [iperf](https://github.com/esnet/iperf) and [netperf](https://hewlettpackard.github.io/netperf/)),
- resource manager-oriented (e.g., measuring VM placement latency and boot time), and
- database-oriented (e.g., [YCSB](https://github.com/brianfrankcooper/YCSB) on [Cassandra](http://cassandra.apache.org/) and [MongoDB](https://www.mongodb.com/), [memtier_benchmark](https://github.com/RedisLabs/memtier_benchmark) on [Redis](https://redis.io/)),


Due to the shortage of time, benchmarks belonging to the latter group still do not run properly.
However, the current bugs seem to be pretty trivial to solve.
For instance, [MongoDB](https://www.mongodb.com/) does not start because of a missing release file:

```
kubemarks-pkb STDERR: E: The repository 'https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/3.0 Release' does not have a Release file.
```

And [Redis](https://redis.io/) simply exceeds [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/):

```                                                                    │
kubemarks-pkb STDERR: Error from server (Forbidden): error when creating "/tmp/tmpaIV4bj": pods "pkb-77f34c4d-9" is forbidden: exceeded quota: default-quota, requested: limits.cpu=5, used: limits.cpu=50, limited: limits.cpu=54      
```

Non-running benchmarks are tracked in [issue #5](https://github.com/marcomicera/kubemarks/issues/5).

Please note that [PerfKit Benchmarker](https://github.com/marcomicera/PerfKitBenchmarker) has to be extended so that benchmarks can expose additional information to the [Prometheus](https://prometheus.io/) [Pushgateway](https://github.com/prometheus/pushgateway) (e.g., [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) IDs).
Besides tracking benchmarks that still have to be extended, [issue #21](https://github.com/marcomicera/kubemarks/issues/21) also lists a few previous commits that show how to do this.

## [PerfKit Benchmarker fork](https://github.com/marcomicera/PerfKitBenchmarker) changes
Besides minor bug fixes, the current [PerfKit Benchmarker](https://github.com/marcomicera/PerfKitBenchmarker) fork has been extended with an additional "results writer", i.e., endpoint to which results are exported at the end of a single benchmark execution.
More specifically, it now includes a [Prometheus](https://prometheus.io/) [Pushgateway](https://github.com/prometheus/pushgateway) exporter, which exposes results according to the [OpenMetrics](https://openmetrics.io/) format.
The [official Prometheus Python client](https://github.com/prometheus/client_python) has been used to implement this result writer.
Furthermore, the CSV writer can now write results in "append" mode, allowing it to gradually add entries to the same CSV file as soon as benchmarks finish.

### Including node IDs in benchmark results
While [PerfKit Benchmarker](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker) does include physical node information in benchmark results (e.g., `lscpu` command output), it does not include [Kubernetes](https://kubernetes.io/) node IDs.
This information is essential to make a [comparison between different hardware solutions](#introduction).
Since [PerfKit Benchmarker](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker) is [in charge of creating and configuring pods](#existing-work), its source code had to be extended to make pods aware of the node ID they were running on.
To do this, the [Kubernetes](https://kubernetes.io/) _[Downward API](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/\#the-downward-api)_ comes in handy: it makes it possible to [expose pod and container fields to a running container](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/\#the-downward-api): here is the JSON snippet which made that possible:

```json
'env': [{
    'name': 'KUBE_NODE',
    'valueFrom': {
        'fieldRef': {
        'fieldPath': 'spec.nodeName'
        }
    }
}]
```

This way, each [Kubernetes](https://kubernetes.io/) pod can retrieve the node ID of the physical machine on which it is running, and [PerfKit Benchmarker](https://github.com/marcomicera/PerfKitBenchmarker) can successfully include this information in the results.

## Running benchmarks periodically
Benchmarks are run periodically as a [Kubernetes](https://kubernetes.io/) _[CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)_.
It periodically executes a shell script ([`scripts/pkb/start.sh`](../scripts/pkb/start.sh)) that cycles through all the benchmarks to be executed and, for each one of them, it
1. checks whether it is compatible with [Kubernetes](https://kubernetes.io/), and
1. builds a proper argument list to be passed to [PerfKit Benchmarker](https://github.com/marcomicera/PerfKitBenchmarker).

### Docker images
A [Kubernetes](https://kubernetes.io/) [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) launches periodic jobs in Docker containers.
The base [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) file of this repository [`yaml/base/kubemarks-pkb-cronjob.yaml`](../yaml/base/kubemarks-pkb-cronjob.yaml) mainly executes [PerfKit Benchmarker](https://github.com/marcomicera/PerfKitBenchmarker), which in turn needs to launch benchmarks in Docker containers so that the [Kubernetes](https://kubernetes.io/) scheduler can allocate those onto pods.
[`marcomicera/kubemarks-cronjob`](https://hub.docker.com/r/marcomicera/kubemarks-cronjob) and [`marcomicera/kubemarks-pkb`](https://hub.docker.com/r/marcomicera/kubemarks-pkb) are the Docker images launched by the [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) and [PerfKit Benchmarker](https://github.com/marcomicera/PerfKitBenchmarker), respectively.

- The former launches [PerfKit Benchmarker](https://github.com/marcomicera/PerfKitBenchmarker) via the [`scripts/pkb/start.sh`](../scripts/pkb/start.sh) script (more info [here](#running-benchmarks-periodically)).
- The latter takes care of resolving most of the dependencies needed by benchmarks so that [PerfKit Benchmarker](https://github.com/marcomicera/PerfKitBenchmarker) will not waste any other time doing so.

### [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) files and their structure
[Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) is a tool included in `kubectl` that allows users to customize [Kubernetes](https://kubernetes.io/) objects. In this project, it is mainly used to:
1. combine multiple objects in the right order of declaration into a single YAML file, and
1. enable object inheritance.

Objects combination is particularly useful when creating the [base CronJob file that runs an user-defined list of benchmarks](#the-base-cronjob-file-running-an-user-defined-benchmarks-list), while inheritance makes it possible to create [dedicated CronJob files for single benchmarks](#dedicated-cronjob-files-for-benchmarks), making it simpler for the user to launch single benchmarks.

#### The [base CronJob file](../yaml/base/kubemarks-pkb-cronjob.yaml) running an user-defined benchmarks list
[Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) needs a [kustomization](https://github.com/kubernetes-sigs/kustomize/blob/master/docs/glossary.md#kustomization) file as a starting point.
The [base `kustomization.yaml` file](../yaml/base/kustomization.yaml) simply lists all the [Kubernetes](https://kubernetes.io/) objects needed to launch [`kubemarks`](https://github.com/marcomicera/kubemarks):

```yaml
resources:
  - kubemarks-conf.yaml # benchmarks list, pushgateway address
  - kubemarks-role.yaml # PerfKit Benchmarker permissions
  - kubemarks-role-binding.yaml # associating Role to ServiceAccount
  - kubemarks-pkb-cronjob.yaml # PerfKit Benchmarker base CronJob file
  - kubemarks-git-creds.yaml # Git credentials for downloading private repo
  - kubemarks-sa.yaml # ServiceAccount
  - kubemarks-num-pods.yaml # number of pods to be used for each benchmark
```

All these files can be combined together and applied with a single command:

```bash
$ kubectl kustomize yaml/base | kubectl apply -f -
```

Before launching this command, [`yaml/base/kubemarks-num-pods.yaml`](../yaml/base/kubemarks-num-pods.yaml), [`yaml/base/kubemarks-conf.yaml`](../yaml/base/kubemarks-conf.yaml), and [`yaml/base/kubemarks-pkb-cronjob.yaml`](../yaml/base/kubemarks-pkb-cronjob.yaml) should be modified accordingly, as described in the [Guide section](#guide).

##### Repo cloner [InitContainer](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
An [InitContainer](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) takes care of cloning this repository first.
If this repository is private, then it needs Git credentials mounted as a [Secret](https://kubernetes.io/docs/concepts/configuration/secret): please take a look at the [corresponding developer guide section](../CONTRIBUTING.md#git-credentials-secret-for-private-repository) to learn more.

#### Dedicated CronJob files for benchmarks
When launching single benchmarks, [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) can be used to override some [Kubernetes](https://kubernetes.io/) object fields of the [base CronJob file](../yaml/base/kubemarks-pkb-cronjob.yaml).
Looking at a benchmark-specific [kustomization](https://github.com/kubernetes-sigs/kustomize/blob/master/docs/glossary.md#kustomization) file is enough to determine which fields are actually overridden.
The following [`kustomization.yaml`](../yaml/benchmarks/fio/kustomization.yaml) file depicts all changes needed to run the [fio](../yaml/benchmarks/fio) benchmark:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../base
nameSuffix: -fio
patchesStrategicMerge:
  - benchmarks-list.yaml
  - schedule.yaml
```

This file inherits all [Kubernetes](https://kubernetes.io/) objects of the [base `kustomization.yaml` file](../yaml/base/kustomization.yaml), and:
- adds a metadata-name suffix to all its [Kubernetes](https://kubernetes.io/) objects (i.e., `-fio`),
- defines the list of benchmarks to be executed (a [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) in [`yaml/benchmarks/fio/benchmarks-list.yaml`](../yaml/benchmarks/fio/benchmarks-list.yaml) containing only `fio` in the list) and,
- overrides the frequency with which this benchmark is run (based on its average completion time).

The [`nameSuffix`](https://kubectl.docs.kubernetes.io/pages/reference/kustomize.html#namesuffix) field causes all resources to have this suffix prepended to their name, hence each benchmark will run in a dedicated [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) whose name contains the benchmark one.

It is worth noticing that since every benchmark's [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) folder (e.g., [`yaml/benchmarks/fio`](../yaml/benchmarks/fio)) contains its own [`schedule.yaml` file](../yaml/benchmarks/fio/schedule.yaml), every benchmark will be launched at different frequencies.
Benchmarks already have pre-determined frequency values based on their average completion time (i.e., the longer they take to complete, the less frequent their corresponding [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) will be launched), but this can be overridden by simply editing their `schedule.yaml` file **before launching**.
Here is an example of [`schedule.yaml` file (for the fio benchmark)](../yaml/benchmarks/fio/schedule.yaml):

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: kubemarks-pkb
spec:
  schedule: '0 */4 * * *'
status: {}
```

According to the [Cron](https://en.wikipedia.org/wiki/Cron) format, fio will be launched every 4 hours.
Modifying such files while a benchmark is running in a [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) will not cause any change.

## Permissions
Upon launching a benchmark, [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) uses the [base `kustomization.yaml` file](../yaml/base/kustomization.yaml) (as described [here](#the-base-cronjob-file-running-an-user-defined-benchmarks-list)) to assemble together various [Kubernetes](https://kubernetes.io/) objects.
Between these objects, there are a couple ones used for [_Role-based access control_](https://kubernetes.io/docs/reference/access-authn-authz/rbac/), such as a [ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) [`yaml/base/kubemarks-sa.yaml`](../yaml/base/kubemarks-sa.yaml), a [Role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings) [`yaml/base/kubemarks-role.yaml`](../yaml/base/kubemarks-role.yaml) and a [RoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings) object [`yaml/base/kubemarks-role-binding.yaml`](../yaml/base/kubemarks-role-binding.yaml).

# User guide
This section examines the [How to run it section](../README.md#how-to-run-it) of the [main `README.md` file](../README.md) in depth.

## Configuration
This section describes all configuration steps to be made before launching benchmarks.

### Number of [Kubernetes](https://kubernetes.io/) pods
The number of [Kubernetes](https://kubernetes.io/) pods to be used for every benchmark is defined in the [`yaml/base/kubemarks-num-pods.yaml`](../yaml/base/kubemarks-num-pods.yaml) [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/). Here is an extract:

```yaml
kind: ConfigMap
metadata:
  name: kubemarks-num-pods
apiVersion: v1
data:
  kubemarks-num-pods.yaml: |
    flags:
      cloud: Kubernetes
      kubernetes_anti_affinity: false

    block_storage_workload:
      description: >
        Runs FIO in sequential, random, read and
        write modes to simulate various scenarios.
      vm_groups:
        default:
          vm_count: 1
```

It is worth noticing that [PerfKit Benchmarker](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker) uses the term _VM_ as a generalization of _[Kubernetes](https://kubernetes.io/) pod_ since it supports multiple cloud providers.

This [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) will be then applied by [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) upon launching a benchmark, and it will then be mounted as a file in the container running [PerfKit Benchmarker](https://github.com/marcomicera/PerfKitBenchmarker).

### [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) frequency
Next, the general [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) frequency can be adjusted in the [`yaml/base/kubemarks-pkb-cronjob.yaml`](../yaml/base/kubemarks-pkb-cronjob.yaml) file:

```yaml
schedule: '*/30 * * * *'
```

The schedule follows the [Cron](https://en.wikipedia.org/wiki/Cron) format.
There is no need to specify this for the [one-benchmark-only CronJob files](#dedicated-cronjob-files-for-benchmarks), as the frequency will be automatically set by [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/).

### Benchmarks list and [Pushgateway](https://github.com/prometheus/pushgateway) address
The [`yaml/base/kubemarks-conf.yaml`](../yaml/base/kubemarks-conf.yaml) file contains two experiment options, namely
- the [Prometheus](https://prometheus.io/) [Pushgateway](https://github.com/prometheus/pushgateway) address, and
- the list of benchmarks to run.

```yaml
# `kubectl kustomize yaml/base | kubectl apply -f -` will
# automatically update this ConfigMap
apiVersion: v1
data:
  benchmarks: cluster_boot fio
  pushgateway: pushgateway.address.test
kind: ConfigMap
metadata:
  name: kubemarks-conf
```

Experiments can be chosen amongst this list:
- `block_storage_workload`
- `cluster_boot`
- `fio`
- `iperf`
- `mesh_network`
- `netperf`
<!--
- `cassandra_ycsb`
- `cassandra_stress`
- `mongodb_ycsb`
- `redis`
-->

This [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) will be automatically applied/updated by [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/).

## Launching benchmarks
Benchmarks can be either launched singularly with (e.g., for [iperf](https://github.com/esnet/iperf)):

```bash
$ kubectl kustomize yaml/benchmarks/iperf | kubectl apply -f -
```

or programmatically, after [having updated the list of benchmarks to run](#benchmarks-list-and-pushgateway-address):

```bash
$ kubectl kustomize yaml/base | kubectl apply -f -
```

# Developer guide
Take a look at the [`CONTRIBUTING.md` file](../CONTRIBUTING.md).

# Conclusions
[`kubemarks`](https://github.com/marcomicera/kubemarks) is able to [periodically](#running-benchmarks-periodically) run [various kinds](#supported-benchmarks) of benchmarks on a [Kubernetes](https://kubernetes.io/) cluster.
The [current PerfKit Benchmarker fork](https://github.com/marcomicera/PerfKitBenchmarker) (more details [here](#perfkit-benchmarker-fork-changes)) includes [physical node identifiers into benchmark results](#including-node-ids-in-benchmark-results) and gradually exposes them to a [Prometheus](https://prometheus.io/) [Pushgateway](https://github.com/prometheus/pushgateway) following the [OpenMetrics](https://openmetrics.io/) format.
The tool is configurable through a few handy [configuration files](#configuration).
