-   [Background](#background)
-   [Motivation](#motivation)
-   [Goals](#goals)
-   [Non-Goals](#non-goals)
-   [Design](#design)
    -   [Container images and deploying FATE cluster on
        Kubernetes](#container-images-and-deploying-fate-cluster-on-kubernetes)
    -   [Custom Resource Definition](#custom-resource-definition)
        -   [Kubefate](#kubefate)
        -   [FATECluster](#fatecluster)
        -   [FateJob](#fatejob)
    -   [Controller design](#controller-design)

_Status_
* 2020-5-14 – Draft v1

## Background
Federated machine learning (FML) is a machine learning setting where many clients (e.g. mobile devices or organizations) collaboratively train a model under the coordination of a central server while keeping the training data decentralized. Only the encrypted mediate parameters exchange between clients with MPC or homomorphic encryption.
![Federated Machine Learning](diagrams/fate-operator-fl.png)

FML has received significant interest recently, because of efficient to solve data silos and data privacy preserving problems. The companies participated in federated machine learning include 4Paradigm, ANT Financial, Data Republic, Google, Huawei, Intel, JD.com, Microsoft, Nvidia, OpenMind, Pingan Technology, Sharemind, Tencent, VMware, Webank etc.

Depending on the differences in features and sample data space, federated machine learning can be classified into horizontally federated machine learning, vertically federated machine learning and federated transfer learning. Horizontally federated machine learning also is called sample-based federated machine learning, means data sets share the same feature space but different in samples. With horizontally federated machine learning, we can gather the relatively small or partial data set into a big one to in-crease the performance of trained models. Vertical federated machine learning is applicable to the cases that two data set with different feature space but share same sample ID. With vertical federated machine learning we can trained a model with attributes from different organizations for a full profile. Vertical federated machine learning is required to redesign most of machine learning algorithms. Federated transfer learning applies to scenarios that two data set with different features space but also different samples. 

[FATE (Federated AI Technology Enabler)](https://fate.fedai.org) is an open source project initialized by Webank, now hosted at the Linux Foundation. FATE is the only open source FML framework to support both horizontal and vertical FML currently. The architecture design of FATE is focus on how to provides FML platform for enterprise, especially financial companies. As contributed by VMware, FATE is only open source FML solution running on Kubernetes with [KubeFATE project](https://github.com/FederatedAI/KubeFATE), and proved an effective solution for FML use case. 

## Motivation
Kubeflow provides a toolset for end-to-end machine learning workflow on Kubernetes. Introducing FML to Kubeflow with FATE, will help the FML users and researchers leverage existed Kubeflow toolkits in their workflow and help them more efficiently building FML solution. 

A FATE-Operator is a start of supporting FML in Kubeflow. This proposal is aimed to defining what FATE operator should look like, and how to apply to Kubeflow.

## Goals
A Kubeflow user should be able to run training using FATE as easily as they can using PyTorch, Tensorflow. This proposal is centered around a Kubernetes Operator for FATE. With the FATE-Operator, a user can:
1.	Provision and manage a FATE cluster;
2.	Submit a FML job to FATE.

This proposal defines the following:
1.	A FATE operator with three CRDs: 
   * FateJob: create a FML job;
   * FateCluster: create a FATE cluster to serve FML jobs;
   * Kubefate: the resource management component of FATE cluster.
2.	Example of full lifecycle to create KubeFATE component, deploy FATE cluster and submit a FML job to created FATE and get the result. Note that, KubeFATE and FATE cluster only need deploying once, and handle multiple jobs.

## Non-Goals
For the scope of this proposal, we won’t be addressing the method of serving the model.

## Design

### Container images and deploying FATE cluster on Kubernetes
We have built a set of images for FATE cluster, and put into: https://hub.docker.com/orgs/federatedai/repositories . All the images are verified by users. 

And there is a provision and management component of FATE cluster, called [KubeFATE](https://github.com/FederatedAI/KubeFATE/tree/master/k8s-deploy). KubeFATE manage FATE clusters in one federated party.

All images work well and are proved in users’ environments.

### Custom Resource Definition
#### Kubefate
```
apiVersion: app.kubefate.net/v1beta1
kind: Kubefate
metadata:
  name: kubefate-sample
  namespace: kube-fate
spec:
  # kubefate image tag
  imageVersion: v1.0.2
  # ingress host
  ingressDomain: kubefate.net
  # kubefate config
  config:
    FATECLOUD_MONGO_USERNAME: "root"
    FATECLOUD_MONGO_PASSWORD: "root"
    FATECLOUD_MONGO_DATABASE: "KubeFate"
    FATECLOUD_REPO_NAME: "kubefate"
    FATECLOUD_REPO_URL: "https://federatedai.github.io/KubeFATE/"
    FATECLOUD_USER_USERNAME: "admin"
    FATECLOUD_USER_PASSWORD: "admin"
    FATECLOUD_LOG_LEVEL: "debug"
```
KubeFATE is a core component to manage and coordinate FATE clusters in one FML party. The above CRD defines the KubeFATE component. 
* ingressDomain defines other components how to access the service ingress of KubeFATE exposed.

#### FATECluster
```
apiVersion: app.kubefate.net/v1beta1
kind: FateCluster
metadata:
  name: fatecluster-sample
  namespace: fate-9999
spec:
  version: v1.4.0
  partyId: "9999"
  proxyPort: "30009"
  partyList:
  - partyId: "10000"
    partyIp: "192.168.1.10"
    partyPort: "30010"
  egg:
replica: 1
  # Points to KubeFATE created by CRD “Kubefate”
  kubefate:
    name: kubefate-sample
    namespace:  kube-fate
```
The FateCluster defines a deployment of FATE on Kubernetes. 
* version defines the FATE version deployed in Kubernetes;
* partyId defines the FML party’s ID;
* proxyPort defines the exposed port for exchanging models and middle variables between different parties in a FML. It will be exposed as a node port;
* partyList defines the parties in a federation which are collaboratively learning;
* egg is the worker nodes of FATE.

#### FateJob
```
apiVersion: app.kubefate.net/v1beta1
kind: FateJob
metadata:
  name: fatejob-sample
  namespace: fate-9999
spec:
  # fateClusterRef points to the FATE cluster resource created by FateCluster
  fateClusterRef: fatecluster-sample
  jobConf:
    pipeline: |-
      {
        "components": {
          "secure_add_example_0": {
            "module": "SecureAddExample"
          }
        }
      }
    modulesConfig: |-
      {
        "initiator": {
          "role": "guest",
          "party_id": 9999
        },
        "job_parameters": {
          "work_mode": 1
        },
        "role": {
          "guest": [
            9999
          ],
          "host": [
            9999
          ]
        },
        "role_parameters": {
          "guest": {
            "secure_add_example_0": {
              "seed": [
                123
              ]
            }
          },
          "host": {
            "secure_add_example_0": {
              "seed": [
                321
              ]
            }
          }
        },
        "algorithm_parameters": {
          "secure_add_example_0": {
            "partition": 10,
            "data_num": 1000
          }
        }
      }
```
FateJob defines the job sent to FATE cluster:
* fateClusterRef defines the cluster of FATE deployed on Kubernetes. Its value is resource name of FATE cluster created by CRD “FateCluster”;
* jobConf defines the details of a FML job. It includes:
   * pipeline: the workflow pipeline of FATE. In FATE, there are many prebuilt algorithm components (ref: https://github.com/FederatedAI/FATE/tree/master/federatedml and https://github.com/FederatedAI/FATE/tree/master/federatedrec), which can be used to train own models. The pipeline defines how data go through and process in the whole training flow;
   * moduleConf: the detail configuration of each algorithm component. E.g. the optimizers, the batch size etc.

### Controller design

We created a new custom controller for FateCluster, FateJob and Kubefate resources. 

The relationship between them and processes to make everything work is shown as following diagrams.

Process 1. Create Kubefate if not existed. The custom controller (1) listens for Kubefate resource, (2) create RBAC resource (Role, Service Account, RoleBinding) to allow remote execution, (3) then create the related resource of Kubefate. (4) The controller waits the resource of Kubefate ready and return the status.

![Process 1](diagrams/fate-operator-1.png)

Process 2. Create FATE cluster. In one federated party, only one Kubefate need to provision and manage infrastructure, connection, status. And multiple FATE cluster which handle jobs of different purples, such as dev cluster, production cluster, collaboration with different external organizations etc. (1) the custom FATE controller listens for FateCluster resource, (2) and call Kubefate cluster existed in the one federated party, send the metadata and configurations (3) to create a FATE cluster. (4) The controller waits the resource of FATE ready and return the status.

![Process 2](diagrams/fate-operator-2.png)

Process 3. Submit FML job to FATE cluster. If FATE cluster existed in one federated party. It is ready for receiving learning job. (1) The custom FATE controller listens for FateJob CRD, and send the job conf to FATE cluster, which includes the pipeline and modules conf. (3) The FATE controller waits for the job result from FATE cluster.  

![Process 3](diagrams/fate-operator-3.png)

The overall architecture of the learning can be presented as following diagram, the FATE cluster will handle the communication and model averaging itself and get back to FATE controller once the job done.

![Overall](diagrams/fate-operator-overall.png)