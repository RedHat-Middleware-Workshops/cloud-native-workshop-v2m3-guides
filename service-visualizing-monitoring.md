## Lab2 - Service Visualization and Monitoring

In this lba, we will visualize your service mesh using **Kiali**, **Prometheus**, **Grafana** and you will lean how to configure basic **Istio funtionalities** such as **RouteRule**, **A/B Testing**.

####1. Generating application load

---

To get a better idea of the power of metrics, let's setup an endless loop that will continually access the application and generate load. We'll open up a separate terminal just for this purpose. 

Open a new _Terminal_ and execute this command with your own _Bookinfo App URL_:

`BOOK_URL=REPLACE WITH YOUR BOOKINFO APP URL`

`while true; do curl -o /dev/null -s -w "%{http_code}\n" http://$BOOK_URL/productpage ; sleep .2 ; done`

This command will endlessly access the application and report the HTTP status result in a separate terminal window.

With this application load running, metrics will become much more interesting in the next few steps.

####2. Examine Kiali

---

**Kiali** is a frontend for Maistra that will allow you to manage and monitor your mesh from a single UI. This UI will allow you to view configurations, monitor traffic flow and health, and analyze traces.

Go to **Networling > Routes** in the left menu and select _istio-system_ in project. Click on Kiali route link.

![istio-kiali]({% image_path istio-kiali-route.png %})

You should see _Kiali Login_ screen. Enter the username and password as below and click _Log In_.

 * Username: `admin`
 * Password: `admin`

![istio-kiali]({% image_path istio-kiali-login.png %})

Input the namespace of your **userxx-bookinfo** application(i.e. user1-bookinfo) and Enter it.

![kiali]({% image_path kiali-all-namespaces.png %})

Then you will only see your working namespace as below:

![kiali]({% image_path kiali-bookinfo-namespaces.png %}){:width="800px"}

##### Service Graph

Go to the Service Graph page on the Graph menu and check **Traffic Animation** in **Display**:

![kiali]({% image_path kiali-service-graph.png %})

It shows a graph with all the microservices, connected by the requests going through then. On this page, you can see how the services interact with each other.


##### Applications

Click on **Applications** menu in the left navigation. On this page you can view a listing of all the services that
are running in the cluster, and additional information about them, such as health status.

![kiali]({% image_path kiali-applications.png %})

Click on the **productpage** application to see its details and you can also see the health of a service (a service is considered healthy)
on the **Health** section when it’s online and responding to requests without errors:

![kiali]({% image_path kiali-app-productpage.png %})

By clicking on **Inbound Metrics**, you can see the metrics for an application, like so:

![kiali]({% image_path kiali-app-productpage-inbound.png %})

By clicking on **Outbound Metrics**, you can see the metrics for an application, like so:

![kiali]({% image_path kiali-app-productpage-outbound.png %})

##### Workloads

Click on **Workloads** menu in the left navigation. On this page you can view a listing of all the workloads are present on your applications.

![kiali]({% image_path kiali-app-productpage-workload.png %})

Click on the **productpage-v1** workload. Here you can see details for the workload, such as the pods and services that are included in it:

![kiali]({% image_path kiali-app-productpage-workload-v1.png %})

By clicking _Inbound Metrics_, you can check the metrics for the workload. The metrics are the same as the Application ones.

##### Services

Click on **Services** menu in the left navigation. Here, you can see the listing of all services.

![kiali]({% image_path kiali-services.png %})

Click on **productpage** service. You can, on this page, see the details of the service, such as metrics, traces, workloads, virtual services, destination rules and so on:

![kiali]({% image_path kiali-services-productpage.png %})

####3. Querying Metrics with Prometheus

---

[Prometheus](https://prometheus.io/) exposes an endpoint serving generated metric values. The Prometheus
add-on is a Prometheus server that comes pre-configured to scrape Mixer endpoints
to collect the exposed metrics. It provides a mechanism for persistent storage
and querying of Istio metrics.

Go to **istio-system** overview page in [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"} and click on the **Prometheus** route link:

![istio-prometheus]({% image_path istio-prometheus-route.png %})

You should see Prometheus home screen, similar to this:

![istio-prometheus]({% image_path istio-prometheus-landing.png %})

In the “Expression” input box at the top of the web page, enter the text: **istio_request_duration_seconds_count**. Then, click the **Execute** button.

You should see a listing of each of the application's services along with a count of how many times it was accessed.

![Prometheus console]({% image_path istio-prometheus-console.png %})

You can also graph the results over time by clicking on the _Graph_ tab (adjust the timeframe from 1 hour to 1 minute for
example):

![Prometheus graph]({% image_path istio-prometheus-graph.png %})

Other expressions to try:

* Total count of all requests to _productpage_ service: **istio_request_duration_seconds_count{destination_service=~"productpage.*"}**
* Total count of all requests to _v3_ of the _reviews_ service: **istio_request_duration_seconds_count{destination_service=~"reviews.*", destination_version="v3"}**
* Rate of requests over the past 5 minutes to all _productpage_ services: **rate(istio_request_duration_seconds_count{destination_service=~"productpage.*", response_code="200"}[5m])**

There are many, many different queries you can perform to extract the data you need. Consult the
[Prometheus documentation](https://prometheus.io/docs) for more detail.

####4. Visualizing Metrics with Grafana

---

As the number of services and interactions grows in your application, this style of metrics may be a bit
overwhelming. [Grafana](https://grafana.com/){:target="_blank"} provides a visual representation of many available Prometheus
metrics extracted from the Istio data plane and can be used to quickly spot problems and take action.

Go to istio-system overview page in [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"} and click on the **Grafana** route link:

![istio-grafana]({% image_path istio-grafana-route.png %})

You should see Grafana home screen, similar to this:

![Grafana graph]({% image_path grafana-home.png %})

##### Istio Mesh Metrics

Select **Home > Istio Mesh Dashboard** to see Istio mesh metrics:

![Grafana graph]({% image_path grafana-mesh-metrics-select.png %})

You will a similar metrics:

![Grafana graph]({% image_path grafana-mesh-metrics.png %})

##### Istio Service Metrics

Let's see detailed metrics of the **Productpage**service. Click on _productpage.userXX-bookinfo.svc.cluster.local_ and the service dashboard will look similar to this:

![Grafana graph]({% image_path grafana-service-metrics.png %})

The Grafana Dashboard for Istio consists of three main sections:

 * _A Global Summary View_ provides high-level summary of HTTP requests flowing through the service mesh.
 * _A Mesh Summary View_ provides slightly more detail than the Global Summary View, allowing per-service filtering and selection.
 * _Individual Services View_ provides metrics about requests and responses for each individual service within the mesh (HTTP and TCP).

Note that _TCP Bandwidth_ metrics are empty, as Bookinfo uses http-based services only. Lower down on this dashboard are metrics for workloads that call this service (labeled "Client Workloads") and for worloads that process requests from the service (labeled _Service Workloads_).

You can switch to a different service or filter metrics by _client-_ and _service-worloads_ by using drop-down lists at the top of the dashboard.

##### Istio Workload Metrics

To switch to workloads dashboard, select **Home > Istio Workload Dashboard** from the drop-down list in the top left corner of the screen.
You should see a screen similar to this:

![Grafana graph]({% image_path grafana-workload-metrics.png %})

This dashboard shows workload’s metrics, and metrics for client- (inbound) and service (outbound) workloads.
You can switch to a different workload, ot filter metrics by inbound or outbound workloads by using drop-down lists at the top of the dashboard.

For more on how to create, configure, and edit dashboards, please see the [Grafana documentation](http://docs.grafana.org/){:target="_blank"}.

As a developer, you can get quite a bit of information from these metrics without doing anything to the application itself. Let's use our new tools in the next section to see the real power of Istio to diagnose and fix issues in applications and make them more resilient and robust.

####5. Request Routing

---

This task shows you how to configure dynamic request routing based on weights and HTTP headers.

_Route rules_ control how requests are routed within an Istio service mesh. Route rules provide:

* _Timeouts_
* _Bounded retries_ with timeout budgets and variable jitter between retries
* _Limits_ on number of concurrent connections and requests to upstream services
* _Active (periodic) health checks_ on each member of the load balancing pool
* _Fine-grained circuit breakers_ (passive health checks) – applied per instance in the load balancing pool

Requests can be routed based on the source and destination, HTTP header fields, and weights associated with individual service versions. For example, a route rule could route requests to different versions of a service.

Together, these features enable the service mesh to tolerate failing nodes and prevent localized failures from cascading instability to other nodes. However, applications must still be designed to deal with failures by taking appropriate fallback actions. For example, when all instances in a load balancing pool have failed, Istio will return HTTP 503. It is the responsibility of the application to implement any fallback logic that is needed to handle the HTTP 503 error code from an upstream service.

If your application already provides some defensive measures (e.g. using [Netflix Hystrix](https://github.com/Netflix/Hystrix){:target="_blank"}), then that's OK. **Istio** is completely transparent to the application. A failure response returned by Istio would not be distinguishable from a failure response returned by the upstream service to which the call was made.

####6. Service Versions

---

Istio introduces the concept of a service version, which is a finer-grained way to subdivide service instances by versions (_v1_, _v2_) or environment (_staging_, _prod_). These variants are not necessarily different API versions: they could be iterative changes to the same service, deployed in different environments (prod, staging, dev, etc.). Common scenarios where this is used include A/B testing or canary rollouts. Istio’s [traffic routing rules](https://istio.io/docs/concepts/traffic-management/rules-configuration.html){:target="_blank"} can refer to service versions to provide additional control over traffic between services.

![Versions]({% image_path versions.png %})

As illustrated in the figure above, clients of a service have no knowledge of different versions of the service. They can continue to access the services using the hostname/IP address of the service. The Envoy sidecar/proxy intercepts and forwards all requests/responses between the client and the service.

####7. VirtualService objects

---

In addition to the usual OpenShift object types like _BuildConfig_, _DeploymentConfig_, _Service_ and _Route_, you also have new object types installed as part of Istio like _RouteRule_. Adding these objects to the running OpenShift cluster is how you configure routing rules for Istio.

Let's install a default route rule because the BookInfo sample deploys 3 versions of the reviews microservice, we need to set a default route. Otherwise if you access the application several times, you’ll notice that sometimes the output contains star ratings. This is because without an explicit default version set, Istio will route requests to all available versions of a service in a random fashion, and anytime you hit _v1_ version you'll get no stars.

Let's create a default set of **virtual services** which will direct all traffic to the _reviews:v1_ service version:

`oc create -f /projects/cloud-native-workshop-v2m3-labs/istio/virtual-service-all-v1.yaml`

You can see this default set of _virtual services_ with:

`oc get virtualservices -o yaml`

There are default _virtual services_ for each service, such as the one that forces all traffic to the _v1_ version of the _reviews_ service:

`oc get virtualservices/reviews -o yaml`

~~~yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  creationTimestamp: "2019-07-02T15:50:36Z"
  generation: 1
  name: reviews
  namespace: userXX-bookinfo
  resourceVersion: "2899673"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/user1-bookinfo/virtualservices/reviews
  uid: 2240fcce-9ce1-11e9-b30a-0af4213cc1b8
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
~~~

Now, access the application again in your web browser using the below link and reload the page several times - you should not see any rating stars since **reviews:v1** does not access the _ratings_ service.

* Bookinfo Application with no rating stars at `http://$BOOK_URL/productpage`

To verify this, open the Grafana Dashboard:

Scroll down to the **ratings** service in _Istio Service Metrics Dashboard_ and notice that the requests coming from the reviews service have stopped:

![Versions]({% image_path ratings-stopped.png %})

####8. A/B Testing with Istio

---

Let's enable the ratings service for a test user named _jason_ by routing productpage traffic to _reviews:v2_ and the others to _reviews:v3_, but only for our test user. Execute:

`oc apply -f /projects/cloud-native-workshop-v2m3-labs/istio/virtual-service-reviews-jason-v2-v3.yaml`

'Tip': Ignore _Warning: oc apply should be used on resource created by either oc create --save-config or oc apply_ message if you got.

Confirm the rule is created:

`oc get virtualservices/reviews -o yaml`


Notice the _match_ element:

~~~yaml
http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
~~~

This says that for any incoming HTTP request that has a cookie set to the _jason_ user to direct traffic to **reviews:v2**.

Now, access the application via your own _Gateway URL_:

`http://YOUR_BOOK_APP_URL/productpage` and click **Sign In** (at the upper right) and sign in with:

* Username: **jason**
* Password: **jason**

> If you get any certificate security exceptions, just accept them and continue. This is due to the use of self-signed certs.

Once you login, refresh a few times - you should always see the black ratings stars coming from **ratings:v2**.

![Ratings for Test User]({% image_path ratings-testuser.png %})

If you **sign out**, you'll return to the **reviews:v3** version which shows red ratings stars.

![Ratings for Test User]({% image_path ratings-signout.png %})

#####Congratulations!

In this lab, you used Istio to send 100% of the traffic to the v1 version of each of the BookInfo services. You then set a rule to selectively send traffic to version v2 of the reviews service based on a header (i.e., a user cookie) in a request.

Once the v2 version has been tested to our satisfaction, we could use Istio to send traffic from all users to v2, optionally in a gradual fashion.
We’ll explore this in the next step.