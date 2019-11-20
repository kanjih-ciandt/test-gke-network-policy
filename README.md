# Example to use Network-Policy in GKE

To be executed after enabled the network policy in the master and nodes.

This example is basic a merge of these tutorials:
* https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer

*  https://docs.projectcalico.org/v3.5/getting-started/kubernetes/tutorials/advanced-policy



**Step 1**: Create 2 namespaces:

*  `kubectl create ns np-demo-kanji`
*  `kubectl create ns np-demo-ciandt`

**Step 2:** Expose your Deployment as a Service internally in both namespace

Create a Deployment using the sample web application container image that listens on a HTTP server on port 8080:

To create the Deployment:

```
kubectl create -f - <<EOF 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: web
  namespace: np-demo-kanji
spec:
  selector:
    matchLabels:
      run: web
  template:
    metadata:
      labels:
        run: web
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:1.0
        imagePullPolicy: IfNotPresent
        name: web
        ports:
        - containerPort: 8080
          protocol: TCP
EOF
```

```
kubectl create -f - <<EOF 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: web
  namespace: np-demo-ciandt
spec:
  selector:
    matchLabels:
      run: web
  template:
    metadata:
      labels:
        run: web
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:1.0
        imagePullPolicy: IfNotPresent
        name: web
        ports:
        - containerPort: 8080
          protocol: TCP
EOF
```
Check if pods is running

* `kubectl get pods -n np-demo-ciandt`
* `kubectl get pods -n np-demo-kanji`

**Step 3:** Expose Deployments as a Service internally

Create a Service resource to make the web deployment reachable within your container cluster.

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: np-demo-kanji
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: web
  type: NodePort 
EOF
```


```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: np-demo-ciandt
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: web
  type: NodePort
EOF
```
When you create a Service of type NodePort with this command, GKE makes your Service available on a randomly- selected high port number (e.g. 32640) on all the nodes in your cluster.

Verify the Service was created and a node port was allocated:

```
kubectl get service web -n np-demo-ciandt
```
Output:
```
NAME   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
web    NodePort   10.139.59.114   <none>        8080:31536/TCP   3m29s
```



```
kubectl get service web -n np-demo-kanji
```
Output:
```
NAME   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
web    NodePort   10.139.56.197   <none>        8080:30060/TCP   5m12s
```
In the sample output above, the node port for the web Service is 31536 and 30060. 

Also, note that there is no external IP allocated for this Service. Since the GKE nodes are not externally accessible by default, creating this Service does not make your application accessible from the Internet.

To make your HTTP(S) web server application publicly accessible, you need to create an Ingress resource.

**Step 4:** Create an Ingress resource

Ingress is a Kubernetes resource that encapsulates a collection of rules and configuration for routing external HTTP(S) traffic to internal services.

On GKE, Ingress is implemented using Cloud Load Balancing. When you create an Ingress in your cluster, GKE creates an HTTP(S) load balancer and configures it to route traffic to your application.

While the Kubernetes Ingress is a beta resource, meaning how you describe the Ingress object is subject to change, the Cloud Load Balancers that GKE provisions to implement the Ingress are production-ready.

The following config file defines an Ingress resource that directs traffic to your web Service:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: basic-ingress
spec:
  backend:
    serviceName: web
    servicePort: 8080
```

To deploy this Ingress resources:

```
kubectl apply -f - <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: basic-ingress
  namespace: np-demo-ciandt
spec:
  backend:
    serviceName: web
    servicePort: 8080
EOF
```

```
kubectl apply -f - <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: basic-ingress
  namespace: np-demo-kanji
spec:
  backend:
    serviceName: web
    servicePort: 8080
EOF
```

Once you deploy this manifest, Kubernetes creates an Ingress resource on your cluster. The Ingress controller running in your cluster is responsible for creating an HTTP(S) Load Balancer to route all external HTTP traffic (on port 80) to the web NodePort Service you exposed.

**Step 5:** Visit your application

Find out the external IPs address of the load balancer serving your application by running:

`kubectl get ingress basic-ingress -n np-demo-kanji`

Output:
```
NAME            HOSTS   ADDRESS          PORTS   AGE
basic-ingress   *       34.98.127.209   80      97s
```

`curl 34.98.127.209`

Output:
```
Hello, world!
Version: 1.0.0
Hostname: web-ddb799d85-kclkf
```

`kubectl get ingress basic-ingress -n np-demo-ciandt`

Output:
```
NAME            HOSTS   ADDRESS        PORTS   AGE
basic-ingress   *       34.102.232.51   80      3m17s
```

`curl 34.102.232.51`

Output:
```
Hello, world!
Version: 1.0.0
Hostname: web-ddb799d85-t87hm
```

**Step 6:** Deny all ingress traffic to np-demo-kanji namespace

To show networking policy working we will deny all ingress from np-demo-kanji namespace

```
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: np-demo-kanji
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Ingress
EOF
```

`curl 34.98.127.209`

Output:
```
<html><head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<title>502 Server Error</title>
</head>
<body text=#000000 bgcolor=#ffffff>
<h1>Error: Server Error</h1>
<h2>The server encountered a temporary error and could not complete your request.<p>Please try again in 30 seconds.</h2>
<h2></h2>
</body></html>
```
Namespace **np-demo-kanji** is not acception ingress anymore but, the namespace **np-demo-ciandt** keep working fine

`curl 34.102.232.51`

Output:
```
Hello, world!
Version: 1.0.0
Hostname: web-ddb799d85-t87hm
```

**Step 7:** Deny all egress traffic to np-demo-ciandt namespace

Open up a second shell session which has kubectl connectivity to the Kubernetes cluster and create a busybox pod to test policy access. This pod will be used throughout this tutorial to test policy access.

```
kubectl run --namespace=np-demo-kanji access --rm -ti --image busybox /bin/sh

```

This should open up a shell session inside the access pod, as shown below.

```
If you don't see a command prompt, try pressing enter.
/ #
```

Open up a third shell session which has kubectl connectivity to the Kubernetes cluster and create a busybox pod to test policy access. This pod will be used throughout this tutorial to test policy access.

```
kubectl run --namespace=np-demo-ciandt access --rm -ti --image busybox /bin/sh

```

This should open up a shell session inside the access pod, as shown below.

```
If you don't see a command prompt, try pressing enter.
/ #
```

In both shell session execute:

`wget www.google.com`

Output:
```
Connecting to www.google.com (172.217.15.68:80)
saving to 'index.html'
index.html           100% |*******************************************************************************************************************************************************| 12641  0:00:00 ETA
'index.html' saved
```

Enable egress isolation on the namespace by deploying a default deny all egress traffic policy.

```
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: np-demo-ciandt
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Egress
EOF
```

Try again to execute `wget www.google.com`. You will see that works to **np-demo-kanji** but not to **np-demo-ciandt**

**Step 8:** Cleanup Namespace

You can clean up after this tutorial by deleting the namespace.

```
kubectl delete ns np-demo-ciandt

kubectl delete ns np-demo-kanjih

```

## Troubleshooting:

* If you are using GKE in a **shared VPC**, probably you have a  GKE service account "**service-[SERVICE_PROJECT_1_NUM]@container-engine-robot.iam.gserviceaccount.com**"  in the project where the VPC was created. So give permission of network admin for this service account.
<code>
gcloud projects add-iam-policy-binding [PROJECT OWNER VPC]  --member=serviceAccount:service-[SERVICE_PROJECT_1_NUM]@container-engine-robot.iam.gserviceaccount.com  --role=roles/compute.networkAdmin
</code>

## Links:

[1] - [Creating a cluster network policy](https://cloud.google.com/kubernetes-engine/docs/how-to/network-policy)

[2] - [Advance policy](https://docs.projectcalico.org/v3.5/getting-started/kubernetes/tutorials/advanced-policy)

[3] - [Video about Ingress Controller](https://www.youtube.com/watch?v=NafjRSqpFxk)

[4] - [Cluster-shared-vpc](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-shared-vpc)

[5] - [Setting up HTTP Load Balancing with Ingress](https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer)