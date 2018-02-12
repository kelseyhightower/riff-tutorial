# Riff Tutorial

The riff tutorial walks you through installing the [riff FaaS platform](https://projectriff.io) and [Istio](https://istio.io/about/intro.html) on [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine). The resulting environment is great for demos and exploring riff and Istio, but should not be considered production ready.

Be sure to watch Mark Fisher's SpringOne [riff announcement keynote and live demo](https://projectriff.io/video/mark-fisher-at-springone-platform-2017/) to get up to speed on riff.

> In addition to the environment Mark demo'd in his keynote, this tutorial integrates Istio and riff to provide traffic management for the riff http gateway and runs each function managed by riff in the Istio service mesh.

## Tutorial

Create a Kubernetes cluster large enough to host the riff and istio components:

```
gcloud container clusters create riff \
  --cluster-version 1.9.2-gke.1 \
  --machine-type n1-standard-4 \
  --num-nodes 5
```

> Smaller clusters should work, but only the configuration above has been tested.

Grant cluster admin permissions to the current user. Admin permissions are required to create the necessary RBAC rules for riff and Istio.

```
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user=$(gcloud config get-value core/account)
```

### Install Istio

Install Istio into the `istio-system` namespace:

```
kubectl apply -f https://raw.githubusercontent.com/kelseyhightower/riff-tutorial/master/istio.yaml
```

> The following addons are also installed: [Prometheus](https://istio.io/docs/tasks/telemetry/metrics-logs.html), [Grafana](https://istio.io/docs/tasks/telemetry/using-istio-dashboard.html), [Jaeger](https://istio.io/docs/tasks/telemetry/distributed-tracing.html), and [Istio Sidecar Injector](https://istio.io/docs/setup/kubernetes/sidecar-injection.html#automatic-sidecar-injection).

Wait until each Istio component is running:

```
kubectl get pods -n istio-system
```
```
NAME                                      READY     STATUS    RESTARTS   AGE
grafana-6585bdf64c-8k45v                  1/1       Running   0          39s
istio-ca-7876b944bc-fx9r9                 1/1       Running   0          40s
istio-ingress-d8d5fdc86-4n6hr             1/1       Running   0          40s
istio-mixer-65bb55df98-b9gwb              3/3       Running   0          46s
istio-pilot-5cb545f47c-ppf6c              2/2       Running   0          40s
istio-sidecar-injector-6bb584c47d-xd6x6   1/1       Running   0          38s
jaeger-deployment-559c8b9b8-d4l9h         1/1       Running   0          39s
prometheus-5db8cc75f8-hxdc5               1/1       Running   0          40s
```

### Install riff

Install riff into the `riff` namespace and enable [Istio automatic sidecar injection](https://istio.io/docs/setup/kubernetes/sidecar-injection.html#deploying-an-app):

```
kubectl apply -f https://raw.githubusercontent.com/kelseyhightower/riff-tutorial/master/riff.yaml
```

Wait until each riff component is running:

```
kubectl get pods -n riff
```
```
NAME                                   READY     STATUS    RESTARTS   AGE
function-controller-54f964dc6c-qv9cf   2/2       Running   0          47s
http-gateway-56d47d5dd-frgb4           2/2       Running   2          46s
kafka-broker-697bbbcbf8-j5mz2          2/2       Running   1          47s
topic-controller-54cbc965bc-nm7rr      2/2       Running   0          46s
zookeeper-77dbfc6cf8-zrlwp             2/2       Running   0          47s
```

> Restarts of the `http-gateway` and `kafka-broker` are expected as kafka depends on zookeeper, and the http-gateway depends on kafka.

### Creating and Executing Fuctions

Ensure the istio sidecar is injected into every riff function created in the default namespace.

```
kubectl label namespace default istio-injection=enabled
```

> riff functions are packaged and deployed as pods which is why the istio injection works.

Create a topic that will trigger the `helloriff` function container:

```
kubectl apply -f https://raw.githubusercontent.com/kelseyhightower/riff-tutorial/master/topics/helloriff.yaml
```

Create the `helloriff` function:

```
kubectl apply -f https://raw.githubusercontent.com/kelseyhightower/riff-tutorial/master/functions/helloriff.yaml
```

> You'll notice we are creating a function without writing any code. riff support using containers as "functions". In this case the `gcr.io/hightowerlabs/helloriff:0.0.1` container will be mapped to events on the `helloriff` topic.

Once the function has been defined the riff `function-controller` will create a deployment with an initial replica count set to zero.

```
kubectl get deployment
```
```
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
helloriff   0         0         0            0           12s
```

riff will scale up the number of pods based on the flow of incoming requests to the riff `http-gateway`.

### Invoking the Function

Retrieve the IP address of the riff `http-gateway` ingress:


```
HTTP_GATEWAY_IP=$(kubectl get ingress http-gateway -n riff \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

Execute an HTTP request to the riff `http-gateway` to invoke the `helloriff` function:


```
curl http://${HTTP_GATEWAY_IP}/requests/helloriff
```

The `curl` command will take a few moments to complete while the `function-controller` scales up the `helloriff` deployment in the background:

```
kubectl get deployments helloriff
```
```
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
helloriff   1         1         1            1           1m
```

Notice the `helloriff` function pod contains three containers: istio-sidecar, riff-sidecar, and the helloriff container defined in the [function definition](https://github.com/kelseyhightower/riff-tutorial/blob/master/functions/helloriff.yaml).

```
kubectl get pods
```
```
NAME                         READY     STATUS    RESTARTS   AGE
helloriff-747b8b5685-fhjf4   3/3       Running   1          11s
```

> The `helloriff` pod restart is expected and maybe related to the interaction between the riff sidecar and kafka.

## Cleanup

Delete all riff resources:

```
kubectl delete ns riff
```

Delete all istio resources:

```
kubectl delete ns istio-system
```

Delete the Kubernetes cluster:

```
gcloud container clusters delete riff
```
