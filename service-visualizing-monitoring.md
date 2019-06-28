## Lab2 - Service Visualization and Monitoring

In this lba, we will visualize your service mesh using **Kiali**, **Prometheus**, **Grafana** and you will lean how to configure
basic **Istio funtionalities** such as **RouteRule8**, **A/B Testing**.

####1. Generating application load

---

To get a better idea of the power of metrics, let's setup an endless loop that will continually access
the application and generate load. We'll open up a separate terminal just for this purpose. 
Open a new **Terminal** and execute this command:

`export GATEWAY_URL=$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}')`

`while true; do curl -o /dev/null -s -w "%{http_code}\n" http://${GATEWAY_URL}/productpage; sleep .2 ; done`

This command will endlessly access the application and report the HTTP status result in a separate terminal window.

With this application load running, metrics will become much more interesting in the next few steps.

####2. Examine Kiali

---

**Kiali** is a frontend for Maistra that will allow you to manage and monitor your mesh from a single UI. This UI will allow you to view configurations, monitor traffic flow and health, and analyze traces.

Access Kiali via finding the exposed route of **istio-system** project:

`oc get route -n istio-system kiali -o jsonpath='{.spec.host}'`

After you have your route, using your web browser navigate to this address using **https**. 
This will bring you to a login screen. Enter the username and password as below and click Log In.

 * Username: **admin**
 * Password: **admin**

![kiali]({% image_path kiali-login.png %})

Input the namespace of your **userxx-bookinfo** application(i.e. user1-bookinfo) and Enter it. 

![kiali]({% image_path kiali-all-namespaces.png %})

Then you will only see your working namespace as below:

![kiali]({% image_path kiali-bookinfo-namespaces.png %}){:width="800px"}

##### Service Graph

Go to the Service Graph page on the Graph menu:

![kiali]({% image_path kiali-service-graph.png %})

It shows a graph with all the microservices, connected by the requests going through then. On this page, 
you can see how the services interact with each other.

##### Applications

Click on **Applications** menu in the left navigation. On this page you can view a listing of all the services that 
are running in the cluster, and additional information about them, such as health status.

![kiali]({% image_path kiali-applications.png %})

Click on the **productpage** application to see its details and you can also see the health of a service (a service is considered healthy) 
on the **Health** section when it’s online and responding to requests without errors:

![kiali]({% image_path kiali-app-productpage.png %})

By clicking on **Inbound Metrics** and change **Source** in **Reported from**, you can also see the metrics for an application, like so:

![kiali]({% image_path kiali-app-productpage-inbound.png %})

##### Workloads

Click on **Workloads** menu in the left navigation. On this page you can view a listing of all the workloads are present on your applications.

![kiali]({% image_path kiali-app-productpage-workload.png %})

Click on the **productpage-v1** workload. Here you can see details for the workload, such as the pods and services that are included in it:

![kiali]({% image_path kiali-app-productpage-workload-v1.png %})

By clicking **Inbound Metrics**, you can check the metrics for the workload. The metrics are the same as the Application ones.

##### Services

Click on **Services** menu in the left navigation. Here, you can see the listing of all services.

![kiali]({% image_path kiali-services.png %})

Click on **productpage** service. You can, on this page, see the details of the service, such as metrics, traces, workloads, virtual services, destination rules and so on:

![kiali]({% image_path kiali-services-productpage.png %})

##### Distributed Tracing

Click on the Distributed Tracing link in the left navigation. The distributed tracing, provided by Jaeger, will open in a new page.

NOTE: The tracing page opens in a new browser window/tab, so if it doesn’t open, please check if your browser didn’t block it from opening.

####3. Querying Metrics with Prometheus

---

[Prometheus](https://prometheus.io/) exposes an endpoint serving generated metric values. The Prometheus
add-on is a Prometheus server that comes pre-configured to scrape Mixer endpoints
to collect the exposed metrics. It provides a mechanism for persistent storage
and querying of Istio metrics.

Go to **istio-system** overview page in OpenShift Web Console and click on the Prometheus route link:

![istio-prometheus]({% image_path istio-prometheus-route.png %})

You should see Prometheus' home screen, similar to this:

![istio-prometheus]({% image_path istio-prometheus-landing.png %})

In the “Expression” input box at the top of the web page, enter the text: `istio_request_duration_seconds_count`.
Then, click the **Execute** button.

You should see a listing of each of the application's services along with a count of how many times it was accessed.

![Prometheus console]({% image_path istio-prometheus-console.png %})

You can also graph the results over time by clicking on the _Graph_ tab (adjust the timeframe from 1 hour to 1 minute for
example):

![Prometheus graph]({% image_path istio-prometheus-graph.png %})

Other expressions to try:

* Total count of all requests to `productpage` service: `istio_request_count{destination_service=~"productpage.*"}`
* Total count of all requests to `v3` of the `reviews` service: `istio_request_count{destination_service=~"reviews.*", destination_version="v3"}`
* Rate of requests over the past 5 minutes to all `productpage` services: `rate(istio_request_count{destination_service=~"productpage.*", response_code="200"}[5m])`

There are many, many different queries you can perform to extract the data you need. Consult the
[Prometheus documentation](https://prometheus.io/docs) for more detail.

####4. Visualizing Metrics with Grafana

---

As the number of services and interactions grows in your application, this style of metrics may be a bit
overwhelming. [Grafana](https://grafana.com/) provides a visual representation of many available Prometheus
metrics extracted from the Istio data plane and can be used to quickly spot problems and take action.

Open the Grafana Dashboard:

* Grafana Dashboard at **http://grafana-istio-system.$ROUTE_SUFFIX/dashboard/db/istio-dashboard**

![Grafana graph]({% image_path grafana-dash.png %})

The Grafana Dashboard for Istio consists of three main sections:

1. **A Global Summary View.** This section provides high-level summary of HTTP requests flowing through the service mesh.
1. **A Mesh Summary View.** This section provides slightly more detail than the Global Summary View, allowing per-service filtering and selection.
1. **Individual Services View.** This section provides metrics about requests and responses for each individual service within the mesh (HTTP and TCP).

Scroll down to the `ratings` service graph:

![Grafana graph]({% image_path grafana-ratings.png %})

This graph shows which other services are accessing the `ratings` service. You can see that
`reviews:v2` and `reviews:v3` are calling the `ratings` service, and each call is resulting in
`HTTP 200 (OK)`. Since the default routing is _round-robin_, that means each reviews service is
calling the ratings service equally. And `reviews:v1` never calls it, as we expect.

For more on how to create, configure, and edit dashboards, please see the [Grafana documentation](http://docs.grafana.org/).

As a developer, you can get quite a bit of information from these metrics without doing anything to the application
itself. Let's use our new tools in the next section to see the real power of Istio to diagnose and fix issues in
applications and make them more resilient and robust.

####5. Request Routing

---

In this step, we will learn how to configure dynamic request routing based on weights and HTTP headers.

_Route rules_ control how requests are routed within an Istio service mesh. Route rules provide:

* **Timeouts**
* **Bounded retries** with timeout budgets and variable jitter between retries
* **Limits** on number of concurrent connections and requests to upstream services
* **Active (periodic) health checks** on each member of the load balancing pool
* **Fine-grained circuit breakers** (passive health checks) – applied per instance in the load balancing pool

Requests can be routed based on
the source and destination, HTTP header fields, and weights associated with individual service versions.
For example, a route rule could route requests to different versions of a service.

Together, these features enable the service mesh to tolerate failing nodes and prevent localized failures from cascading instability to other nodes.
However, applications must still be designed to deal with failures by taking appropriate fallback actions.
For example, when all instances in a load balancing pool have failed, Istio will return HTTP 503. It is
the responsibility of the application to implement any fallback logic that is needed to handle the HTTP
503 error code from an upstream service.

If your application already provides some defensive measures (e.g. using [Netflix Hystrix](https://github.com/Netflix/Hystrix)), then that's OK:
Istio is completely transparent to the application. A failure response returned by Istio would not be
distinguishable from a failure response returned by the upstream service to which the call was made.

####6. Service Versions

---

Istio introduces the concept of a service version, which is a finer-grained way to subdivide
service instances by versions (`v1`, `v2`) or environment (`staging`, `prod`). These variants are not
necessarily different API versions: they could be iterative changes to the same service, deployed
in different environments (prod, staging, dev, etc.). Common scenarios where this is used include
A/B testing or canary rollouts. Istio’s [traffic routing rules](https://istio.io/docs/concepts/traffic-management/rules-configuration.html) can refer to service versions to
provide additional control over traffic between services.

![Versions]({% image_path versions.png %})

As illustrated in the figure above, clients of a service have no knowledge of different versions of the service. They can continue to access the services using the hostname/IP address of the service. The Envoy sidecar/proxy intercepts and forwards all requests/responses between the client and the service.

####7. RouteRule objects

---

In addition to the usual OpenShift object types like `BuildConfig`, `DeploymentConfig`, `Service` and `Route`,
you also have new object types installed as part of Istio like `RouteRule`. Adding these objects to the running
OpenShift cluster is how you configure routing rules for Istio.

Let's install a default route rule because the BookInfo sample deploys 3 versions of the reviews microservice, we need to set a default route.
Otherwise if you access the application several times, you’ll notice that sometimes the output contains star
ratings. This is because without an explicit default version set, Istio will route requests to all available
versions of a service in a random fashion, and anytime you hit `v1` version you'll get no stars.

First, let's set an environment variable to point to Istio:

~~~shell
export ISTIO_VERSION=0.6.0; 
export ISTIO_HOME=${HOME}/istio-${ISTIO_VERSION}; 
export PATH=${PATH}:${ISTIO_HOME}/bin; 
cd ${ISTIO_HOME}`
~~~

Now let's install a default set of routing rules which will direct all traffic to the `reviews:v1` service version:

~~~shell
oc create -f samples/bookinfo/kube/route-rule-all-v1.yaml
~~~

You can see this default set of rules with:

~~~shell
oc get routerules -o yaml
~~~

There are default routing rules for each service, such as the one that forces all traffic to the `v1` version of the `reviews` service:

~~~shell
oc get routerules/reviews-default -o yaml
~~~

~~~yaml
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: reviews-default
  namespace: default
  ...
spec:
  destination:
    name: reviews
  precedence: 1
  route:
  - labels:
      version: v1
~~~

Now, access the application again in your browser using the below link and reload the page several times - you should not see any rating stars since `reviews:v1` does not access the `ratings` service.

* Bookinfo Application with no rating stars at **http://istio-ingress-istio-system.$ROUTE_SUFFIX/productpage**

To verify this, open the Grafana Dashboard:

* Grafana Dashboard at **http://grafana-istio-system.$ROUTE_SUFFIX/dashboard/db/istio-dashboard**

Scroll down to the `ratings` service and notice that the requests coming from the reviews service have stopped:

![Versions]({% image_path ratings-stopped.png %})

####8. A/B Testing with Istio

---

Lets enable the ratings service for a test user named “jason” by routing **productpage** traffic to **reviews:v2**, but only for our test user. Execute:

~~~shell
oc create -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
~~~

Confirm the rule is created:

~~~shell
oc get routerule reviews-test-v2 -o yaml
~~~

Notice the **match** element:

~~~yaml
  match:
    request:
      headers:
        cookie:
          regex: ^(.*?;)?(user=jason)(;.*)?$
~~~

This says that for any incoming HTTP request that has a cookie set to the `jason` user to direct traffic to
`reviews:v2`.

Now, access the application at 

**http://istio-ingress-istio-system.$ROUTE_SUFFIX/productpage)** and click **Sign In** (at the upper right) and sign in with:

* Username: `jason`
* Password: `jason`

> If you get any certificate security exceptions, just accept them and continue. This is due to the use of self-signed certs.

Once you login, refresh a few times - you should always see the black ratings stars coming from `ratings:v2`. If you logout,
you'll return to the `reviews:v1` version which shows no stars. You may even see a small blip of access to `ratings:v2` on the
Grafana dashboard if you refresh quickly 5-10 times while logged in as the test user `jason`.

![Ratings for Test User]({% image_path ratings-testuser.png %})

#####Congratulations!