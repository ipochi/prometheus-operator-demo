Unless you've been living under a rock, you've probably heard about <a href="https://kubernetes.io/" target="_blank" rel="noopener">Kubernetes</a>, an open-source container orchestration platform designed to automate the deployment, scaling, and management of containerized applications. In fact, Kubernetes has established itself as the de facto standard for container orchestration and is the flagship project of the Cloud Native Computing Foundation(<a href="https://www.cncf.io/" target="_blank" rel="noopener">CNCF</a>)

Kubernetes is great, solves the pain points of application deployment and maintenance in a distributed system but what makes it awesome is its extensibility. Operators is one such powerful concept that makes use of this capability.

In this post we will take a dive into Prometheus operator, how to install and of course using Prometheus operator to monitor our <a href="https://github.com/microservices-demo/microservices-demo" target="_blank" rel="noopener">demo application</a>.
<h2 style="text-align: justify;">Introduction</h2>
Introduced in 2016 by <a href="https://coreos.com/" target="_blank" rel="noopener">CoreOS</a>, an Operator is a method of packaging, deploying and managing a Kubernetes application. A Kubernetes application is an application that is both deployed on Kubernetes and managed using the Kubernetes APIs and kubectl tooling.

In another words an Operator is an application specific custom controller that works directly with kubernetes API as such it can create,configure,manage instances of complex stateful applications according to the custom rules written inside our custom controller.

Before we go any further into the details, lets dial it back a little to understand two important concepts which bring Operators to life.
<ul>
 	<li><a href="https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#custom-resources" target="_blank" rel="noopener">Custom Resource</a></li>
 	<li><a href="https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#custom-controllers" target="_blank" rel="noopener">Custom Controller</a></li>
</ul>
A <strong>Custom Resource</strong> is an object that extends the Kubernetes API or allows you to introduce your own API into a kubernetes cluster.

A <strong>Custom controller</strong> handles in-built Kubernetes objects, such as Deployment, Service in new ways, or manage custom resources as if they were a native kubernetes components.

Naturally custom controllers are most effective when combined with custom resources and operator pattern is one such combination.
<h2 style="text-align: justify;">A bit more about Operator</h2>
As said earlier, an operator builds upon the two central kubernetes concepts <strong>Resource</strong> and <strong>Controller</strong> and adds a set of knowledge or configuration that allows the operator to execute common application tasks. They ultimately help you focus on a desired configuration, not the details of manual deployment and life cycle management.

Let's look at it from an example:

When scaling an etcd cluster manually, a user has to perform a number of steps:
<ul>
 	<li>create a DNS name for the new etcd member</li>
 	<li>launch the new etcd instance</li>
 	<li>use the etcd administrative tools (etcdctl member add) to tell the existing cluster about this new member</li>
</ul>
Instead with the etcd Operator a user can simply increase etcd cluster size field by 1.
<h2 style="text-align: justify;">Operator use cases</h2>
A Kubernetes Operator can:
<ul>
 	<li>Install and provide initial configuration and sizing for your deployment</li>
 	<li>Perform live reloading for any user-requested parameter modification (hot config reloading)</li>
 	<li>Automatically scale up or down according to performance metrics</li>
 	<li>Perform backups, integrity checks or any other maintenance task</li>
</ul>
<h2 style="text-align: justify;">Prometheus Operator</h2>
Getting a kubernetes cluster <a href="http://blog.infracloud.io/ha-onpremise-kubernetes-kubespray/" target="_blank" rel="noopener">up and running</a> is very easy these days, but when you start deploying applications you are bound to run into some issues, coupled with the fact that Kubernetes being a distributed system makes troubleshooting not so easy.

Recently graduated from CNCF <a href="https://prometheus.io/" target="_blank" rel="noopener">Prometheus</a> has become the standard tool for monitoring and alerting in Kubernetes and container world.It provides by far the most detailed and actionable metrics and analysis.

<a href="https://github.com/coreos/prometheus-operator" target="_blank" rel="noopener">Prometheus-operator</a> is a CoreOS conception that provides easy monitoring definitions for Kubernetes services, deployment and management of Prometheus instances.

Once deployed, Prometheus Operator provides the following features:
<ul>
 	<li>Create/Destroy: Easily launch a Prometheus instance for your Kubernetes namespace, a specific application or team easily using the Operator</li>
 	<li>Simple Configuration: Configure the fundamentals of Prometheus like versions, persistence, retention policies, and replicas from a native Kubernetes resource</li>
 	<li>Target Services via Labels: Automatically generate monitoring target configurations based on familiar Kubernetes label queries; no need to learn a Prometheus specific configuration language</li>
</ul>
<h2 style="text-align: justify;">How it works?</h2>
The main idea behind is to decouple the deployment of Prometheus instances from the configuration of the entities they are monitoring.

To implement this functionality, prometheus operator introduces additional resources and abstractions as <a href="https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions" target="_blank" rel="noopener">Custom Resource Definitions</a>(CRD)
<ul>
 	<li><strong>Prometheus</strong> : Defines a desired Prometheus deployment</li>
 	<li><strong>ServiceMonitor</strong> : Specifies how group of services are to be monitored with hte help of labels. Similar to how Services monitor endpoints</li>
 	<li><strong>AlertManager</strong> : Defines a desired AlertManager deployment</li>
 	<li><strong>PrometheusRule</strong>: Defines a desired Prometheus rule file, which can be loaded by a Prometheus instance containing Prometheus alerting and recording rules.</li>
</ul>

| ![prometheus-operator.png](https://github.com/ipochi/prometheus-operator-demo/raw/master/architecture.png) | 
|:--:| 
| *Image Credits: CoreOS* |

From the picture above you can see that you can create a ServiceMonitor resource which will scrape the Prometheus metrics from the defined set of pods. Basically, the Operator instructs Prometheus to watch over the kubernetes API and upon detecting changes, creates a new set of configuration for the new service.
<h2 style="text-align: justify;">Enough theory, let's deploy</h2>
To install the prometheus operator (<strong>prometheus-operator:v0.25.0</strong>), lets start by applying the manifests one by one and explain the reasoning behind it them.

To get this started , you'll need a kubernetes cluster you have access to, also the following set of deployments makes the assumption that RBAC is enabled on your cluster.
<pre>$ git clone git@github.com:ipochi/prometheus-operator-demo.git
$ cd prometheus-operator-demo
</pre>
Below action deploys the prometheus operator, its ClusterRole, ClusterRoleBinding and the ServiceAccount.It grants the Prometheus Operator the following cluster-wide permissions:
<ul>
 	<li>read access to pods, nodes, and namespaces.</li>
 	<li>read/write access to services and their endpoints.</li>
 	<li>full access to secrets, ConfigMaps , StatefulSets, Prometheus-related resources (alert managers, service monitors,etc)</li>
</ul>
<pre>$ kubectl apply -f manifests/prometheus-operator.yaml
</pre>
Check if the operator is successfully deployed or not. You should see the output similar to below
<pre>$ kubectl get crd
NAME                                    CREATED AT
alertmanagers.monitoring.coreos.com     2018-11-29T17:38:34Z
prometheuses.monitoring.coreos.com      2018-11-29T17:38:34Z
prometheusrules.monitoring.coreos.com   2018-11-29T17:38:35Z
servicemonitors.monitoring.coreos.com   2018-11-29T17:38:35Z


$ kubectl get pods -n default
NAME                                  READY     STATUS    RESTARTS   AGE
prometheus-operator-c4b75f7cd-w28kw   1/1       Running   0          18m

$  kubectl get serviceaccount | grep prometheus
prometheus            1         21m
prometheus-operator   1         21m

$ kubectl get clusterrolebinding | grep prometheus
prometheus                                             22m
prometheus-operator                                    22m

$ kubectl get clusterrole | grep prometheus
prometheus                                                             22m
prometheus-operator                                                    23m
</pre>
Next up is the ClusterRole and ClusterRoleBinding for the Prometheus Pods

Assuming that RBAC authorization is activated, we need to create RBAC rules for both Prometheus and Prometheus Operator. A ClusterRole and a ClusterRoleBinding for the Prometheus Operator were created in the first step.The same must be done for the Prometheus Pods.

The below manifest creates a ClusterRole and ClusterRoleBinding for the Prometheus pods
<pre>$ kubectl apply -f manifests/prometheus-pods-rbac.yaml
</pre>
We have deployed Prometheus operator, respective CRDs and corresponding ClusterRole and ClusterRoleBindings for both operator and its pods.

There are couple of things left to deploy namely the Prometheus Pods, ServiceMonitors for providing the scraping configuration and Service to expose Prometheus onto a specific node port, but first we will deploy our demo application and we will get to the remaining parts as we go along.
<h2 style="text-align: justify;">Monitor Demo application:</h2>
We will be installing <strong>microservices-demo</strong> application from weaveworks.More about the application can be found <a href="https://github.com/microservices-demo">here</a>

Apply the manifests for deploying the application
<pre>$ kubectl create namespace sock-shop
$ kubectl apply -f https://raw.githubusercontent.com/microservices-demo/microservices-demo/master/deploy/kubernetes/complete-demo.yaml
</pre>
Check if application is deployed and we can can access the front end service on the port 30001.
<pre>$ kubectl get svc,deployments -n sock-shop
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/carts          ClusterIP   10.102.153.164                 80/TCP         24m
service/carts-db       ClusterIP   10.104.239.239                 27017/TCP      24m
service/catalogue      ClusterIP   10.109.119.32                  80/TCP         24m
service/catalogue-db   ClusterIP   10.106.130.138                 3306/TCP       24m
service/front-end      NodePort    10.99.139.102                  80:30001/TCP   24m
service/orders         ClusterIP   10.102.230.115                 80/TCP         24m
service/orders-db      ClusterIP   10.101.241.231                 27017/TCP      24m
service/payment        ClusterIP   10.106.171.190                 80/TCP         24m
service/queue-master   ClusterIP   10.106.16.104                  80/TCP         24m
service/rabbitmq       ClusterIP   10.106.177.36                  5672/TCP       24m
service/shipping       ClusterIP   10.98.16.105                   80/TCP         24m
service/user           ClusterIP   10.109.22.162                  80/TCP         24m
service/user-db        ClusterIP   10.98.190.154                  27017/TCP      24m

NAME                                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/carts          1         1         1            1           24m
deployment.extensions/carts-db       1         1         1            1           24m
deployment.extensions/catalogue      1         1         1            1           24m
deployment.extensions/catalogue-db   1         1         1            1           24m
deployment.extensions/front-end      1         1         1            1           24m
deployment.extensions/orders         1         1         1            1           24m
deployment.extensions/orders-db      1         1         1            1           24m
deployment.extensions/payment        1         1         1            1           24m
deployment.extensions/queue-master   1         1         1            1           24m
deployment.extensions/rabbitmq       1         1         1            1           24m
deployment.extensions/shipping       1         1         1            1           24m
deployment.extensions/user           1         1         1            1           24m
deployment.extensions/user-db        1         1         1            1           24m
</pre>
All services in our demo application expose metrics via the <strong>/metrics</strong> endpoint, this information needs to be provided to Prometheus pods so that it can scrape our metrics. ServiceMonitors are just the thing that we need for this task.

Lets create ServiceMonitor for our front-end service. Our front-end service exposes the metrics at containerPort 8079 via /metrics path.

https://gist.github.com/ipochi/a110c3e916a357cf86cf6f572c5d7b26

ServiceMonitor works in the same manner of label selection as Pods and Services. In the above yaml we are specifying the selector to match the label <strong>front-end</strong> against all the services that are present in the namespace <strong>sock-shop</strong> and our target being container port 8079 and path <strong>/metrics</strong>

Similarly you'll find that we have created other servicemonitors for the remaining services in our application. Lets apply them all.
<pre>$ kubectl apply -f manifests/service-monitors/</pre>
To verify that everything is working correctly.
<pre>$ kubectl get servicemonitors -n sock-shop
NAME                  AGE
sock-shop-carts       26m
sock-shop-catalogue   26m
sock-shop-front-end   26m
sock-shop-orders      26m
sock-shop-payment     26m
sock-shop-shipping    26m
sock-shop-user        26m
</pre>
Now, onto creation of Prometheus pod manifest in which we will provide the information as to which ServiceMonitors it needs to pick for scraping.

https://gist.github.com/ipochi/98ab7c98303a38bdb029f55d67f82555

<pre>$ kubectl apply -f manifests/prometheus.yaml
</pre>
This tells Prometheus pods to scrap from those ServiceMonitor's with the labels [front-end, carts, catalogue, orders, payment, shipping, user]

Lets verify if our prometheus pods are up and running in the default namespace.
<pre>$ kubectl get pods -n default
NAME                                  READY     STATUS    RESTARTS   AGE
prometheus-operator-c4b75f7cd-w28kw   1/1       Running   0          28m
prometheus-prometheus-0               3/3       Running   1          26m
prometheus-prometheus-1               3/3       Running   2          26m
</pre>
Only thing that's left to do here, is to expose our Prometheus pods via NodePort service, below yaml does just that.

https://gist.github.com/ipochi/72bf27a59b5bcea45ee2691e5f9de31f

<pre>$ kubectl apply -f manifests/service.yaml
</pre>
To confirm our Prometheus pods are scraping the services, lets point our browser to :30900

Navigate to the Status dropdown and select <strong>Targets</strong>. Our services should be listed there.

<img class="alignnone size-large wp-image-1224" src="https://www.infracloud.io/wp-content/uploads/2018/12/prometheus-targets-1024x492.png" alt="" width="1024" height="492" />
<h2 style="text-align: justify;">Operator vs Helm</h2>
There is a small overlap with Helm as both perform application setup.

Helm is a package manager, a good way to organize applications (deployment, service etc. templates packaged into one tar). An analogy for helm would be like 'apt' for Kubernetes.

Operators enable you to manage the operation of applications within Kubernetes using custom resources and controllers.A Helm chart by comparison is a way to template out K8s objects to make them configurable for different environments.

As such both of them are complimentary, lifecycle management of an Operator can be done using kubectl or Helm. you can use helm charts to deploy an operator as well.
<h2 style="text-align: justify;">Conclusions</h2>
Application monitoring is an important part of our application stack, with the help of Prometheus operator, we were able to implement our application monitoring with less effort, in a more declarative and reproducible manner, which is easier to scale,modify or migrate to different set of hosts.