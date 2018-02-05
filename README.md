# K8-Scalar
For this K8-Scalar 101, we will go over the steps to implement and evaluate elastic scaling policies in container-orchestrated database clusters using the Advanced Riemann-Based Autoscaler (ARBA). Furthermore, additional details about infrastructure and operation are appended. 

# Evaluating autoscalers for container-orchestrated database clusters
This tutorial provides more practical know-how for the related paper. Nine steps allow us to effectively implement and evaluate elastic scaling strategies for specific database and workload types.

The setup of a Kubernetes cluster depends on the underlying platform. The _infrastructure_ section provides some references to get started. If you just want to try out the tutorial on your local machine, then you can run directly the bash scripts that are provided by this tutorial. This tutorial installs [MiniKube](https://kubernetes.io/docs/tasks/tools/install-minikube/). 


## Prerequisites



**System requirements**
  * Your local machine should support VT-x virtualization
  * One local VM with minimmaly 4 virtual CPU cores and 8GB virtual memory must be able to run on your machine in order to run 2 Cassandra instances of each 4GB memory. 
 
**Disclaimer**
This local minikube-based setup is not suitable for running scientific experiments. If you want accurate results for the example experiment on a single machine, your could try a minikube VM of 16 virtual CPU cores and 32 GB of virtual memory. But we have never tested this. Moreover Minikube only supports kubernetes clusters with one worker node (i.e. the minikube VM). it is better to run the different components of the K8-Scalar architecture on different VMs as illustrated in Section 3 of the related paper. See the __Infrastructure__ section at the end of this README fule for some advice on how to control the placement of Pods across VMs.  

**Install git:** 
  * MacOS: https://git-scm.com/download/mac
  * Linux Debian: sudo apt-get install git
  * Linux CentOS: sudo yum install git
  * Windows: https://git-scm.com/download/win. GitBash is by default also installed. Open a GitBash session and keep it open during the rest of the experiment
 
**Clone the K8-scalar GitHub repository and set the k8_scalar_dir variable:** 
  
```bash
git clone https://github.com/k8-scalar/k8-scalar/ && export k8_scalar_dir=`pwd`/k8-scalar
```


**Setup other environment variables:**


Note1: This tutorial contains many code snippets for MacOS, Linux or Windows. They are only tested on MacOS and Windows
Note2: The snippets contain environment variables which should be self-declarative. Do not forget to specify them:
  * `${k8_scalar_dir}` = the local directory on your system in which the k8-scalar GitHub project has been cloned
                         This environment variable is already set by executing the above snippet of the git clone
  * `${MyRepository}` = the name of the Docker repository of the customized experiment-controller image (based on Scalar). Create in 
                        advance an account for the ${MyRepository} repository at https://hub.docker.com/. In the context of this tutorial, 
                        and all experiments from the paper are stored in the t138 repository at docker hub.
  * `${my_experiment}` = the name of the directory under `${k8_scalar_dir}` where code and data of your current experiment is stored

  
## (1) Setup a Kubernetes cluster, Helm and install the Heapster monitoring service

[Helm](https://github.com/kubernetes/helm) is utilised to deploy the distributed system of the experiment. Helm is a package manager for Kubernetes charts. These charts are packages of pre-configured Kubernetes resources. For this system, we provide three charts. A shared _monitoring-core_ is used across several experiments. This core contains _Heapster_, _Grafana_ and _InfluxDb_. The second chart provides a database cluster and the third the ARBA system with an experiment controller included.


### For Mac OS:
install kubectl, minikube and helm client
```bash
# Install VirtualBox
brew cask install virtualbox
# Install kubectl
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/darwin/amd64/kubectl && chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/kubectl
# Install MiniKube
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.24.1/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
# Start MiniKube
minikube start --cpus 4 --memory 8192

```
On windows 10 we experienced a bootstrapping race-condition:
We get an authorization error when running `kubectl get nodes` while all appropriate credentials are actually correctly configured:

```
$ kubectl.exe get nodes
error: You must be logged in to the server (Unauthorized)
```
What helps is to wait some time, switch to the minikube kubectl context, and
open the ./minikube/config file

```
$ kubectl config use-context minikube
Switched to context "minikube".

$vi ./.minikube/config

$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     <none>    21m       v1.9.0
```

Install Helm
```
# Install Helm client:
curl -LO https://kubernetes-helm.storage.googleapis.com/helm-v2.8.0-darwin-amd64.tar.gz && tar xvzf helm-v2.8.0-darwin-amd64.tar.gz && chmod +x ./darwin-amd64/helm && sudo mv ./darwin-amd64/helm /usr/local/bin/helm
# Install Helm server on Kubernetes cluster
helm init
```
### For Linux on bare-metal:

Install [VirtualBox](https://www.virtualbox.org/wiki/Linux_Downloads)

install kubectl, minikube and helm client
```bash
# Install kubectl:
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kubectl && chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/kubectl
#Install MiniKube
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
# Start MiniKube with enough resources
minikube start --cpus 4 --memory 8192
```

If you get an authorization error when running `kubectl get nodes`:
```
$ kubectl.exe get nodes
error: You must be logged in to the server (Unauthorized)
```
then you have to switch to the minikube kubectl context

```
$ kubectl config use-context minikube
Switched to context "minikube".

$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     <none>    21m       v1.9.0
```

Install Helm
```
# Install Helm client
curl -LO https://kubernetes-helm.storage.googleapis.com/helm-v2.8.0-linux-amd64.tar.gz && tar xvzf helm-v2.8.0-linux-amd64.tar.gz && chmod +x ./linux-amd64/helm && sudo mv ./linux-amd64/helm /usr/local/bin/helm
# Install Helm server on Kubernetes cluster
helm init
```

### For Windows:

Install [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

Open the GitBash desktop application

install kubectl, minikube and helm client
```bash
# Install kubectl
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/windows/amd64/kubectl.exe && export PATH=$PATH:`pwd`

# Install minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-windows-amd64.exe && mv minikube-windows-amd64.exe minikube.exe && export PATH=$PATH:`pwd`
minikube start --cpus 4 --memory 8192
```
It takes several minutes on our Windows 10 machine before the Kubernetes worker node gets ready.
```
$ kubectl.exe get nodes
error: You must be logged in to the server (Unauthorized)
...

$ kubectl get nodes
NAME       STATUS     ROLES     AGE       VERSION
minikube   NotReady   <none>    4d        v1.9.0

...

$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     <none>    21m       v1.9.0
```

Install Helm
```
# Install Helm client
curl -LO https://kubernetes-helm.storage.googleapis.com/helm-v2.8.0-windows-amd64.tar.gz && tar xvzf helm-v2.8.0-windows-amd64.tar.gz && export PATH=$PATH:`pwd`/windows-amd64/
# Install Helm server on Kubernetes cluster
helm init
```

## Deploy Heapster monitoring service
Now with Kubernetes and Helm installed, you should be able to install services on Kubernetes using Helm.

Let us add the monitoring capabilities of Heapstwe to the cluster using Helm. We install the _monitoring-core_ chart by the following command. This chart includes instantiated templates for the following Kubernetes services: Heapster, Grafana and the InfluxDb. 
```bash
helm install ${k8_scalar_dir}/operations/monitoring-core
```

To check if all services are running execute the following command to see if all Pods of these services are running
```bash
$ kubectl get pods --namespace=kube-system
NAME                                    READY     STATUS    RESTARTS   AGE
heapster-76647b5d6c-ln7lp               1/1       Running   0          6m
kube-addon-manager-minikube             1/1       Running   0          17m
kube-dns-54cccfbdf8-wstwt               3/3       Running   0          17m
kubernetes-dashboard-77d8b98585-qqkmx   1/1       Running   0          17m
monitoring-grafana-8fcc5f8d6-x49wv      1/1       Running   0          6m
monitoring-influxdb-7bf9b74f99-kpvr8    1/1       Running   0          6m
storage-provisioner                     1/1       Running   0          17m
tiller-deploy-7594bf7b76-598xv          1/1       Running   0          7m
```



## (2) Setup a Database Cluster
This _cassandra-cluster_ chart uses a modified image which resolves a missing dependency in one of Google Cassandra's image. Of course, this chart can be replaced with a different database technology. Do mind that Scalar will have to be modified for the experiment with implementations of desired workload generators for the Cassandra database. The next step will provide more information about this modification.
```bash 
helm install ${k8_scalar_dir}/operations/cassandra-cluster
```

## (3) Determine and implement desired workload type for the deployed database in Scalar
This step requires some custom development for different database technologies. Extend Scalar with custom _users_ for your database which can read, write or perform more complex operations. For more information how to implement this, we refer to the [Cassandra User classes](development/scalar/src/be/kuleuven/distrinet/scalar/users) and the [Cassandra Request classes](development/scalar/src/be/kuleuven/distrinet/scalar/requests). Afterwards we want to build the application and copy the resulting jar:
```bash
# Extend User with operations for your database in the directory below
cd ${k8_scalar_dir}/development/scalar/src/be/kuleuven/distrinet/scalar/users
vim ${myDatabase}User.java # Cfr CassandraWriteUser.java

# After building the project, copy the resulting Jar file to the experiment-controller image
# Note source code of scalar is not yet included in this project. You have to build from the existing scalar-1-0-0.jar and use the resulting # jar file.
mv ${k8_scalar_dir}/development/scalar/target/scalar-1-0-0.jar ${k8_scalar_dir}/development/example-experiment/lib/scalar-1-0-0.jar
```
Next we want to configure scalar itself. If you want to configure a linearly increasing workload profile, you don't need to do anything here. The `stress.sh` script offers a user-friendly tool for configuring such workload profile (See step 5 for more detail)

If you want to configure another kind of workload profile, like an oscillating workload profile, you'll need to define this workload profile in the file [experiment.properties](development/scalar/conf/experiment.properties). An explanation overview of all the configuration options for defining a Scalar experiment is explained [here](docs/scalar/features.md).
```
# Configure the experiment-controller's workload
cd ${k8_scalar_dir}/development/example-experiment/etc/
vim experiment-template.properties # Configure user_implementations, do not modify user_implementations_prestart
#to quit vi type ":q <enter>"
``` 

Finally, build a new image for the experiment-controller
```bash
docker build -t ${myRepository}/experiment-controller ${k8_scalar_dir}/development/example-experiment/
# overwrite in following command MyRepository_DOCKERHUB_PASSWRD with your secret password: 
# docker login -u ${myRepository} -p MyRepository_DOCKERHUB_PASSWRD  
docker push ${myRepository}/experiment-controller
```

Scalar is a fully distributed, extensible load testing tool with a numerous features. I recommend checking out https://distrinet.cs.kuleuven.be/software/scalar/ for more information.

## (4) Deploying experiment-controller
Before deploying, check out the [Operations section](README.md#operations) below in this document for performing the necessary Kubernetes secret management and resource configuration. The secret management is mandatory as the experiment-controller requires this to communicate with the Master API of the Kubernetes cluster.

The example experiment in the operations directory contains only a helm chart for deploying the experiment-controller with ARBA. As such the script below illustrates how to directly install the experiment controller using `kubectl`. In this simple exeriment we only deploy a single Pod.

```bash
kubectl create -f ${k8_scalar_dir}/operations/arba-with-experiment-controller/templates/experiment-controller.yaml 
```

## (5) Perform experiment for determining the mapping between SLA violations and resource usage metrics  
Before starting the experiment, we recommend using the `kubectl get pods --all-namespaces` command to validate that no error occured during the deployment. Finally, we can start the experiment by executing the following command:

```bash
kubectl exec experiment-controller -- bash bin/stress.sh --duration 400 500:1500:1000
```
This command will tell Scalar to gradually increase the workload on the database cluster. The workload is executed as a series of runs. The duration of a single run is set at 400 seconds. The workload starts at a run of 500 requests per second and increases up to 1500 with an increment of 1000 requests per second. For these arguments, the experiment will consist thus of 2 runs and last 800 seconds.

The user guide of `stress.sh` is returned when running stress.sh without any option:

```
kubectl exec experiment-controller -- bash bin/stress.sh
bin/stress.sh [OPTIONS] [ARGS]

 Performs an experiment to determine the threshhold to start scaling databases.

 Parameters:
   userload                             Specify user load or, the start user load, end user load and incrementing interval. (Format:  NN or NN:NN:NN)

 Options:
   -d | --duration              Specify the duration in seconds for each run. (Default: 60)
   -p | --pod                   The initial pod (Default: cassandra-0)
   -h | --help                  Display this message

 Note:
   This script MUST be executed from within the experiment directory.

 Examples:
   # Stress Cassandra with an user load of 125 requests per seconds for 200 seconds
   bin/stress.sh --duration 200 125

   # Executes the experiment: 10 users for first run, 20 users for second run, .., 100 users for last run.
   bin/stress.sh 10:100:10
```

Afterwards, experiment results include Scalar statistics and Grafana graphs. The Scalar results are found in the pod experiment-controller pod in the `/exp/var` directory. The Kubernetes cluster exposes a Grafana dashboard at port 30345. Some default graphs are provided, but you can also write your own queries. This snipper provides an easy way to copy the results to the local developer machine. Of course, the second script to open grafana is only valid when trying out the flow on MiniKube. For realistic clusters, you should determine the IP of any Kubernetes node.
```bash
# Open a shell to experiment-controller pod's Scalar results
$ kubectl exec -it experiment-controller -- bash
root@experiment-controller:/exp/var# cd results/
root@experiment-controller:/exp/var/results# ls
run-1500.dat  run-500.dat
root@experiment-controller:/exp/var/results# vi run-500.dat
Experiment consisting of 1 runs and 1 request types:
        CassandraWriteRequest


Load,          Duration,      Troughput,     Capacity,      Slow requests, Timing accuracy
------------------------------------------------------------------------------------------
500,           60s,           21466,         357.767,       1.141%,        21.944ms


Breakdown of residence times for CassandraWriteRequest requests:

Load,          Min,           Max,           Mean,          Std. dev.,     Shape,         Scale          (successful)   (failed)       (error)        (conn_problem) (timed_out)    (redirected)   (no_result)
500            2.166          7756.775       309.948        695.622        0.199          1561.196       21507          0              0              0              0              0              0


Percentiles of sampled residence times (only an approximation if residence_times_sample_fraction is set):

CassandraWriteRequest (run 1, 500 users):
        50.0%   of requests handled in 142.039ms.
        90.0%   of requests handled in 613.618ms.
        95.0%   of requests handled in 958.478ms.
        99.0%   of requests handled in 5273.487ms.
        99.9%   of requests handled in 6851.968ms.
        99.99%  of requests handled in 7756.198ms.
```

```bash
# Open the Grafana dashboard in your default browser and take relevant screenshots
$ minikube ip
192.168.99.100
open http://192.168.99.100:30345/
```




## (6) Implement an elastic scaling policy that monitors the resource usage
This step requires some custom development in the Riemann component. Extend Riemann's configuration with a custom scaling strategy. We recommend checking out http://riemann.io/ to get familiar with the way that events are processed. While Riemann has a slight learning curve, the configuration has access to a Clojure, which is a complete programming language. While out of scope for the provided examplar, new strategies should most often combine events of the same deployment or statefulset by folding them. The image should be build and uploaded to the repository in a similar fashion as demonstrated in step (3).

This example experiment has created an [ARBA image with three scaling strategies](development/riemann/etc/riemann.config) as defined in Table 1 of the related paper. The ARBA service will be deployed with as configuration to use one of these strategies (i.e Strategy 2 of Table 1: `scale if CPU usage > 67% of CPU usage limit`). See [operations/arba-with-experiment-controller/values.yaml](operations/arba-with-experiment-controller/values.yaml).

## (7) Deploy ARBA 

To deploy the ARBA image together with the experiment-controller using Helm, execute the following script

```bash
#kubectl delete old Pod of experiment-controller if not already done
kubectl delete pod experiment-controller
helm install ${k8_scalar_dir}/operations/arba-with-experiment-controller
```

## (8) Test the elastic scaling policy by executing the workload and measuring the number of SLA violations
Finally, this step is very similar to the fifth step. The biggest difference occurs during processing the results. We will analyze the request latencies to determine how many SLO violations have occured with respect to the expected latency of 150 ms. The implemented scaling policy is ineffective if SLO violations have occurred for more than 5% of the requests.

Repeat the experiment:
```bash
kubectl exec experiment-controller -- bash bin/stress.sh --duration 400 500:1500:1000
```
And open grafana to see when a second cassandra Pod is added.
```bash
# Open the Grafana dashboard in your default browser and take relevant screenshots
open http://$(minikube ip):30345/
```
or requests the number of replicas of the statefulset:
```bash
kubectl get statefulset cassandra
```

## (9) Repeat steps 7 and 8 until you have found an elastic scaling policy that works for this workload

# Infrastructure
You need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. For example, create a Kubernetes cluster on Amazon Web Services [(tutorial)](https://kubernetes.io/docs/getting-started-guides/aws/) or quickly bootstrap a best-practice cluster using the [kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/) toolkit. To install helm in distributed cluster, you'll first need to first create a [service-account for Helm](http://jayunit100.blogspot.be/2017/07/helm-on.html) and initiate helm with this service account. Moreover, to install the monitoring system in kubeadm, you need to install the monitoring-core-rbac chart instead of the monitoring-core chart.

```
kubectl create -f ${k8_scalar_dir}/development/helm/helm.yaml
helm init --service-account helm
```

```
helm install ${k8_scalar_dir}/operations/monitoring-core-rbac
```


In this tutorial, however, we will use a MiniKube deployment on our local device.
This is just for demonstrating purposes as the resources provided by a single laptop are unsufficient for valid experiment results.
You can, however, follow the same exact steps on a multi-node cluster.
For a more accurate reproduction scenario, we suggest adding labels to each node and add them as constraints to the YAML files of the relevant Kubernetes objects via a [nodeSelector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector). As such different Kubernetes objects such as experiment-controller and the cassandra instance will not be created on the same node, as presented in the related paper.

# Operations
## Experiment configuration  
The experiment has a mandatory configuration to allow communication with the cluster, and an optional configuration to fine-tune experiment parameters. Also, do not forget to use your own repository name in Kubernetes resource declaration files when uploading custom images.

The autoscaler interacts directly with the Kubernetes cluster. The _kubectl_ tool, which is used for this interaction, requires configuration. Secrets are used to pass this sensitive information to the required pods. The next snippet creates the required keys for a MiniKube cluster. First, prepare a directory that contains all the required files. 

```bash
mkdir ${k8_scalar_dir}/operations/secrets
cd ${k8_scalar_dir}/operations/secrets
cp ~/.kube/config .
cp ~/.minikube/client.crt .
cp ~/.minikube/client.key .
cp ~/.minikube/ca.crt .
vim config
#to quit vim type ":q <enter>"
```

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority: C:\Users\eddy\.minikube\ca.crt
    server: https://192.168.99.102:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    as-user-extra: {}
    client-certificate: C:\Users\eddy\.minikube\client.crt
    client-key: C:\Users\eddy\.minikube\client.key
```


Secondly, change all absolute paths in the  `config` file to the location at which these secrets are mounted by the `arba-with-experiment-controller` Helm chart, i.e. `/root/.kube`. Have a look at the example config file above. The `ca.crt` certificate and the `client.crt` and `client.key` are stored in the `C:\Users\eddy\.minikube` directory of the local machine. This must be changed to `/root/.kube`. You can either do it manually or modify and execute one of the following two sed scripts:

**Windows**
```
#Espacing a backslash requires three backslashes in Windows Cygwin 
sed -ie "s@C:\\\Users\\\eddy\\\.minikube\\\@/root/.kube/@g" ./config
```

**Linux/MacOS**
```
sed -ie "s@/Users/wito/.minikube/@/root/.kube/@g" ./config
```


Finally, the following command will create the secret. You will have to create the same secret in two different namespaces.  Do note that the keys required depend on the platform that you have your cluster deployed on.

```
#Secret for ARBA that runs in the kube-system namespace
kubectl create secret generic kubeconfig --from-file . --namespace=kube-system

#The same secret for the experiment-controller that runs in default namespace
kubectl create secret generic kubeconfig --from-file . --namespace=default
```

## Configuration of other Kubernetes objects 
Several Kubernetes resources can optionally be fine-tuned. Application configuration is done by setting environment variables. For example, the Riemann component can have a strategy configured or the Cassandra cpu threshold at which it should scale. 

Finally, the resource requests and limits of the Cassandra pod can also be adjusted. These files can be found in the `operations` subdirectory, e.g. the Cassandra YAML file can be found in [operations/cassandra-cluster/templates](operations/cassandra-cluster/templates/cassandra-statefulset.yaml). For this MiniKube tutorial we have set for resource Requests lower than the resource limits in comparison to the configuration of Cassandra instances in the [scientifically evaluated experiments of the associated paper](experiments/LMaaS)

# Development
The goal of this section is to explain how to modify the K8-Scalar examplar to experiment with other types of autoscalers and other types of services.

In order to replace the default ARBA autoscaler with another autoscaler,
it is important to understand that the stable parts of the
current implementation are Kubernetes and Heapster. Heapster
can store the collected metrics into different backends, which are
referred to as sinks. Heapster currently supports [16 different sink
types](https://github.com/kubernetes/heapster/blob/master/docs/sink-configuration.md), including the Riemann sink. To plug-in another autoscaler,
the autoscaler should be compatible with one of these sink
types.

The REST-based interface of the Scaler service of ARBA also
aims to offer a portable specification of scaling actions for any type
of application and any type of container orchestration framework.

Finally, Scalar’s implementation can be easily extended with
additional functionality.  It is of course also possible to use another
benchmarking tool as Experiment-Controller if that tool is more appropriate
for a particular experiment. The ARBA-with-experiment-
Controller must then be changed with a Docker image for that
tool.

