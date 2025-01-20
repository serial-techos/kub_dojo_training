
# Write raw Kubernetes manifests



During this exercise, you will learn how to:



- write Kubernetes resource manifests

- create resources based on those manifests



The goal of this exercise is to learn how to deploy applications inside

Kubernetes. You will not be using existing manifests written by the open-source

community. In practice:

- You will write the manifests for your own applications.

- You will usually use customizable existing manifest to deploy open-source software.



Try to use kubectl explain to obtain information on the structure of Kubernetes resources. Asking Google or ChatGPT for copy-pastable YAML will work most of the time, but will not help you understand the underlying concepts that you can grasp with trial and error.



That being said: don't stay stuck, don't hesitate to ask for help, and most

importantly have fun!



## Step 1: Install the tools you will need



For this exercise, you will need



- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl), the official CLI to interact with Kubernetes

- [kind](https://kubernetes.io/docs/tasks/tools/#kind), a tool to create Kubernetes clusters on your local machine using Docker



> [!IMPORTANT]

> In order for _Kind_ to work properly, you often need at least 8GB of RAM.

> If you encounter performance trouble using this tool, ask your teacher to provide you with a remote VM.

> A [dedicated Terraform module](https://github.com/padok-team/terraform-aws-training-vm) exists for this!

> You'll need to adapt some instructions, such as the URLs to access your app.



## Step 2: Bootstrap the environment



> [!IMPORTANT]

> Make sure Docker is running on your computer. Otherwise, none of this will

> work.



Create a local Kubernetes cluster with the exercise ready:bash

./scripts/bootstrap.sh





While it runs, you can read what it does. What is the Kubernetes version installed?

Which tools are included?



## Step 3: Explore the environment



You now have a Kubernetes cluster running on your machine. Have a look at what

is inside:bash

kubectl get pods -A





An ingress controller is deployed to the ingress-nginx namespace. It will

receive all requests you send to a *.localhost hostname. Try it out:bash

curl foo.localhost

# or

http foo.localhost





You should get a 404. This is because there are no Ingress resources in your

cluster. We'll change that later.



## Step 4: Understand the target architecture



Your goal is to deploy an application with a simple event-driven architecture:



1. Users send requests to the producer.

2. The producer pushes items to a queue stored in a Redis instance.

3. A consumer pops items from the queue.

4. The consumer does some work for each item.



Alongside the application, you will deploy a basic monitoring stack that will

continuously measure the number of items in the queue waiting to be processed.



This is what the target architecture looks like:



![Target Architecture](./diagrams/target-architecture.drawio.png)



## Step 5: Deploy the consumer



Start with the simplest service: the consumer. Here is what you need to do:



1. Build a container image based on the ./consumer directory and make the

image available to your cluster.

2. Write a manifest for a Deployment that runs the consumer. The Deployment

should have 3 replicas.

3. Write a manifest for a Service that distributes requests accross the

replicas.

4. Create the Deployment and Service and make sure the Pods were created

successfully.

5. Check the consumers' logs: does the error you see make sense?



Ready? Set. Go!



<details>

<summary>Best practice n°1</summary>


Use the kind load command to avoid pushing your container image to a registry. Be sure to read the command help section!

You should avoid using the tag latest which is the default when you don't specify a tag.

It can lead to unexpected behavior, especially with kind (<https://kind.sigs.k8s.io/docs/user/quick-start/#loading-an-image-into-your-cluster>)



</details>



<details>

<summary>Best practice n°2</summary>


Use kubectl explain deployments.spec to see the documentation for Deployment

resources as your Kubernetes API understands them.



</details>



<details>

<summary>Best practice n°3</summary>



The kubectl logs command has a --selector flag; try to use it.



</details>



<details>

<summary>Solution</summary>


You can find the complete solution in solution/, but don't spoil yourself too much! Take the time to discuss your solution with your peers.



</details>



## Step 6: Deploy the producer



Deploy the second microservice: the producer. Here is what you need to do:



1. Build a container image based on the ./producer directory and make the

image available to your cluster.

2. Write a manifest for a Deployment that runs the producer. The Deployment

should have 3 replicas.

3. Write a manifest for a Service that distributes requests accross the

replicas.

4. Write a manifest for an Ingress that allows you to send requests to the

producer at the producer.localhost hostname.

5. Create the Deployment, Service, and Ingress.

6. Publish messages by sending a **POST request** to

<http://producer.localhost/publish/10>.

7. Check the producers logs for an error about them not finding Redis.



You can do it!



_Best practices from step 5 apply to this step as well. Here are some new ones._



<details>

<summary>Best practice n°1</summary>



Name your ports in your Pods and Services. This makes references to them from

other resources more obvious. Aim for your code to be obvious.



</details>



<details>

<summary>Best practice n°2</summary>


Give the consumer and producer at least one label in common and annother that

they do not share. This makes it easy to select the entire application or a

single microservice.



</details>



<details>

<summary>Best practice n°3</summary>


In your Ingress, only expose endpoints that need to be public. In the case of

the producer, only /publish needs to be exposed. Its /healthz endpoint is

meant for internal use only.



</details>



<details>

<summary>Solution</summary>


You can find the complete solution in solution/, but don't spoil yourself too much! Take the time to discuss your solution with your peers.



</details>



## Step 7: Deploy Redis



Time to get the application working by deploying the queue. Here is what you

need to do:



1. Write a manifest for a StatefulSet that runs a single Redis instance.

2. Use the redis:7.0 container image.

3. Persist Redis's /data directory.

4. Create the StatefulSet.

5. Update the consumer and producer with the Redis server's address.

6. Publish a few messages with the producers.

7. Check the consumers' logs to see that they processed the messages.



What are you waiting for?



<details>

<summary>Best practice n°1</summary>



It's best to configure services with environment variables, but flags and

configuration files are good as well.



</details>



<details>

<summary>Best practice n°2</summary>


When you can, use a StatefulSet to create your PersistentVolumeClaim, and that

PersistentVolumeClaim to create your PersistentVolume.



</details>



<details>

<summary>Best practice n°3</summary>


Redis has several [way to persist](https://redis.io/docs/manual/persistence/) its data.

Which one is the safest if we don't want to lose any data?



</details>



<details>

<summary>Solution</summary>


You can find the complete solution in solution/, but don't spoil yourself too much! Take the time to discuss your solution with your peers.



</details>



## Step 8: Scale the microservices



The producer and consumer are entirely stateless. This makes them easy to scale

horizontally. Here is what you need to do:



1. Scale the number of producers to 1 replica. Do this with kubectl first,

then make the change permanent in your manifests.

2. Configure the consumer to burn CPU for 100 milliseconds for each message it

processes (👀 read the code if there is an argument for that).

3. Edit the consumer's Pod spec to request 1 CPU core per instance.

4. Write a manifest for a HorizontalPodAutoscaler for the consumer that scales

based on CPU usage between 1 and 3 replicas, with a target average of 70%.

5. Create the HorizontalPodAutoscaler. Check that the number of consumers goes

down to 1.

6. Publish 300 messages. Check that the number of consumers goes up to 3.

7. Check that the number of consumers goes back to 1 when all messages have been

processed.

8. Don't scale the number of Redis replicas. Why is this not a good idea?



Get to it!



<details>

<summary>Best practice n°1</summary>



It's best to keep the smallest number of replicas we can that fits our business

objective.



</details>



<details>

<summary>Best practice n°2</summary>


When scaling to a large number of replicas, be careful not to overload your

cluster.



</details>



<details>

<summary>Best practice n°3</summary>


Always specify resource requests for your Pods. You can make this mandatory in

your cluster or Namespace with a ResourceQuota or LimitRange.



</details>



<details>

<summary>Solution</summary>


You can find the complete solution in solution/, but don't spoil yourself too much! Take the time to discuss your solution with your peers.



</details>



## Step 9: Deploy the Prometheus Redis exporter



Now that the application works as expected, we would like to measure the number

of items in our queue. Use the Prometheus Redis exporter to take measurements.

Here is what you need to do:



1. Create a Deployment to deploy the Prometheus Redis exporter. Use the

oliver006/redis_exporter:v1.54.0 container image.

2. Create a ConfigMap in which you set REDIS_ADDR and

REDIS_EXPORTER_CHECK_SINGLE_KEYS. Use the ConfigMap to define the Pods'

environment variables.

3. Create a Service so Prometheus can scrape the exporter later.

4. Check to make sure the exporter exposes metrics as expected.



Are you ready? Go!



<details>

<summary>Best practice n°1</summary>



Collect metrics for all your services if you can. Measuring everything is a good

way to ensure you know how your application is doing.



</details>



<details>

<summary>Best practice n°2</summary>


When putting environment variables in a ConfigMap, the keys don't necessarily

have to be the name of the variables. This is especially true when the values

are used by multiple services and you want to use a shared ConfigMap.



</details>



<details>

<summary>Best practice n°3</summary>



Use kubectl port-forward to send requests to internal services. This way, you

can make sure your network configuration works as expected, even without an

Ingress.



</details>



<details>

<summary>Solution</summary>


You can find the complete solution in solution/, but don't spoil yourself too much! Take the time to discuss your solution with your peers.



</details>



## Step 10: Deploy Prometheus



Prometheus can scrape metrics from the exporter and from our microservices,

store those metrics, and allow us to query them later. Here is what you need to

do:



1. So far you have probably been deploying to the default namespace. Create a

new namespace called observability. This is where you will deploy

Prometheus and Grafana.

2. Create a ConfigMap containing Prometheus' configuration file. Use the example

below as a basis. Replace the targets with the correct URLs.



yaml

global:

scrape_interval: 5s

scrape_timeout: 1s



scrape_configs:

- job_name: prometheus

static_configs:

- targets:

- localhost:9090

- job_name: producer

static_configs:

- targets:

- producer:8080

- job_name: consumer

static_configs:

- targets:

- consumer:8080

- job_name: redis

static_configs:

- targets:

- redis-exporter:9121





3. Create a StatefulSet to deploy Prometheus.

4. Use the prom/prometheus:v2.47.0 container image.

5. Persist everything Prometheus writes to the /prometheus directory.

6. Create an Ingress so you can access the Prometheus UI at

<http://prometheus.localhost/>.



Are you up to the challenge?



<details>

<summary>Best practice n°1</summary>



You may specify the namespace field in your manifests. It removes the need for the

caller to specify it with kubectl and allows creating resources in multiple

namespaces at once. However, this field can always be overwritten with kubectl if needed.

To reuse easily your manifests, you can omit this field, unless a resource needs to be

in a very specific namespace, such as a ServiceMonitor.



</details>



<details>

<summary>Best practice n°2</summary>


Be careful when writing YAML configuration files inside a ConfigMap manifest.

Knowing which part is parsed as an object and which part is parsed as a string

can be confusing.



</details>



<details>

<summary>Best practice n°3</summary>



Use kubectl's --dry-run and --output flags to generate manifests quickly:bash

kubectl create configmap \

--dry-run=client \

--output=yaml \

--from-file=config.txt





</details>



<details>

<summary>Solution</summary>


You can find the complete solution in solution/, but don't spoil yourself too much! Take the time to discuss your solution with your peers.



</details>



## Step 11: Deploy Grafana



1. Create a StatefulSet to deploy Grafana.

2. Use the grafana/grafana:9.5.10 container image.

3. Persist the contents of /var/lib/grafana.

4. Use a ConfigMap to set the GF_AUTH_ANONYMOUS_ENABLED=true and

GF_AUTH_ANONYMOUS_ORG_ROLE=Admin environment variables.

5. Use a ConfigMap to mount the contents of the ./solution/grafana/provisioning/datasources

directory to /etc/grafana/provisioning/datasources.

6. Use a ConfigMap to mount the contents of the ./solution/grafana/provisioning/dashboards

directory to /etc/grafana/provisioning/dashboards.

7. Use a ConfigMap to mount the contents of the ./solution/grafana/dashboards

directory to /var/lib/grafana/dashboards.

8. Create a Service and an Ingress to access the Grafana UI at

<http://grafana.localhost/>.

9. Check that you have a functioning dashboard for your Redis instance.



You're almost done!



<details>

<summary>Best practice n°1</summary>



Split a service's configuration into multiple ConfigMaps when it makes more

sense than having a single ConfigMap. Services can also share the same

ConfigMap. Do whatever makes your code easier to understand and maintain.



</details>



<details>

<summary>Best practice n°2</summary>


When writing configuration files inside ConfigMaps, keep the raw configuration

files somewhere in your repository and generate the ConfigMap with kubectl.



</details>



<details>

<summary>Best practice n°3</summary>


Use kubectl's --dry-run and --output flags to generate manifests from

entire directories:bash

kubectl create configmap \

--dry-run=client \

--output=yaml \

--from-file=./my-config-dir/





</details>



<details>

<summary>Solution</summary>


You can find the complete solution in solution/, but don't spoil yourself too much! Take the time to discuss your solution with your peers.



</details>



## Cleanup



Once you are done with this exercise, you can destroy your local environment:bash

./scripts/teardown.sh