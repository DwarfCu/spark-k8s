# [Running Spark on Kubernetes](https://spark.apache.org/docs/latest/running-on-kubernetes.html)

> Spark can run on clusters managed by Kubernetes. This feature makes use of native Kubernetes scheduler that has been added to Spark.

> The Kubernetes scheduler is currently experimental. In future versions, there may be behavioral changes around configuration, container images and entrypoints.

## Prerequisites

### A running Kubernetes cluster at version +1.6

```bash
kubectl cluster-info
Kubernetes master is running at https://slik8sm01/clusters/c-8bhgq
KubeDNS is running at https://slik8sm01/k8s/clusters/c-8bhgq/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

### A runnable distribution of Spark +2.3

1. Build Apache Spark Docker image (e.g. Apache Spark 2.4.3).

```bash
# Download a pre-built Spark
cd examples/Spark/
wget http://archive.apache.org/dist/spark/spark-2.4.3/spark-2.4.3-bin-hadoop2.7.tgz

# Extract Spark
tar xzf spark-2.4.3-bin-hadoop2.7.tgz

# Build Spark Docker image
cd spark-2.4.3-bin-hadoop2.7
./bin/docker-image-tool.sh -r dwarfcu -t 2.4.3 build
```

2. Upload Spark Docker image to a repository (e.g. DockerHub).

```bash
docker login

docker push dwarfcu/spark:2.4.3
```

## Submitting Applications to Kubernetes

### Cluster Mode

1. Create Namespace, Service Account and Role Binding.

```bash
kubectl create -f 00-env.yaml
```

2. Submit Spark application (Pi example).

```bash
kubectl run spark --image=dwarfcu/spark:2.4.3 -it -n spark --serviceaccount=spark-sa --restart='Never' --rm=true \
 -- /opt/spark/bin/spark-submit \
 --name spark-pi \
 --master k8s://https://slik8sm01:6443 \
 --deploy-mode cluster \
 --class org.apache.spark.examples.SparkPi \
 --conf spark.kubernetes.driver.pod.name=pi-driver-1 \
 --conf spark.kubernetes.namespace=spark \
 --conf spark.executor.instances=3 \
 --conf spark.kubernetes.container.image=dwarfcu/spark:2.4.3 \
 --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark-sa \
 --conf spark.kubernetes.authenticate.submission.oauthTokenFile=/var/run/secrets/kubernetes.io/serviceaccount/token \
 --conf spark.kubernetes.authenticate.submission.caCertFile=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
 local:///opt/spark/examples/jars/spark-examples_2.11-2.4.3.jar 10000
```

*Notice that in the above example it specifies a jar with a specific URI with a scheme of local://. This URI is the location of the example jar that is already in the Docker image.*

3. Check logs.

```bash
kubectl logs -n spark pi-driver-1
...
Pi is roughly 3.141641147141641
...
```

4. Delete pi-driver-1 pod.

```bash
kubectl delete pod -n spark pi-driver-1
```

5. Delete Namespace, Service Account and Role Binding.

```bash
kubectl delete -f 00-env.yaml
```

### Submitting a personal Spark (JAVA) project

1. Compile a Spark (JAVA) project, e.g. WordCount.

```bash
cd wordCount
mvn clean package
cd ..
```

2. Create Namespace, Service Account and Role Binding. 

```bash
kubectl create -f 00-env.yaml
```

3. Create a ConfigMap from a binary file, i.e. from the JAR file.

```bash
kubectl create configmap -n spark wordcount-jar  --from-file=wordCount/target/wordCount-1.0.jar
```

4. Create a pod for submitting the Spark (JAVA) application (ConfigMaps/Volumes are also mounted).
```bash
kubectl create -f 20-spark-wordCount.yaml
```

5. Delete environment.

```bash
kubectl delete pod -n spark wordCount-driver-1

kubectl delete -f 20-spark-wordCount.yaml

kubectl delete -f 00-evn.yaml
```

Or you can delete only the spark namespace and then all contained resources will also be deleted.

```bash
kubectl delete namespaces spark
```