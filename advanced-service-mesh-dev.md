## Lab3 - Advanced Service Mesh Development

In this lab, you used Istio to send 100% of the traffic to the v1 version of each of the BookInfo services.
You then set a rule to selectively send traffic to version v2 of the reviews service based on a header
(i.e., a user cookie) in a request.

Once the v2 version has been tested to our satisfaction, we could use Istio to send traffic from all users to
v2, optionally in a gradual fashion. We’ll explore this in the next step.


#### Fault Injection

---

This step shows how to inject faults and test the resiliency of your application.

Istio provides a set of failure recovery features that can be taken advantage of by the services
in an application. Features include:

* Timeouts
* Bounded retries with timeout budgets and variable jitter between retries
* Limits on number of concurrent connections and requests to upstream services
* Active (periodic) health checks on each member of the load balancing pool
* Fine-grained circuit breakers (passive health checks) – applied per instance in the load balancing pool

These features can be dynamically configured at runtime through Istio’s traffic management rules.

A combination of active and passive health checks minimizes the chances of accessing an unhealthy service.
When combined with platform-level health checks (such as readiness/liveness probes in OpenShift), applications
can ensure that unhealthy pods/containers/VMs can be quickly weeded out of the service mesh, minimizing the
request failures and impact on latency.

Together, these features enable the service mesh to tolerate failing nodes and prevent localized failures
from cascading instability to other nodes.

While Istio provides a host of failure recovery mechanisms outlined above, it is still imperative to test the
end-to-end failure recovery capability of the application as a whole. Misconfigured failure
recovery policies (e.g., incompatible/restrictive timeouts across service calls) could result
in continued unavailability of critical services in the application, resulting in poor user experience.

Istio enables protocol-specific fault injection into the network (instead of killing pods) by
delaying or corrupting packets at TCP layer.

Two types of faults can be injected: delays and aborts. Delays are timing failures, mimicking
increased network latency, or an overloaded upstream service. Aborts are crash failures that
mimic failures in upstream services. Aborts usually manifest in the form of HTTP error codes,
or TCP connection failures.

####1. Inject a fault

---

To test our application microservices for resiliency, we will inject a 7 second delay between the
`reviews:v2` and `ratings` microservices, for user `jason`. This will be a simulated bug in the code which
we will discover later.

Since the `reviews:v2` service has a
built-in 10 second timeout for its calls to the ratings service, we expect the end-to-end flow
to continue without any errors. Execute:

`oc create -f samples/bookinfo/kube/route-rule-ratings-test-delay.yaml`

And confirm that the delay rule was created:

`oc get routerule ratings-test-delay -o yaml`

Notice the `httpFault` element:

~~~yaml
  httpFault:
    delay:
      fixedDelay: 7.000s
      percent: 100
~~~

Now, access the application at 

`http://istio-ingress-istio-system.$ROUTE_SUFFIX/productpage` and click **Login** and login with:

* Username: `jason`
* Password: `jason`

If the application’s front page was set to correctly handle delays, we expect it to load within
approximately 7 seconds. To see the web page response times, open the Developer Tools menu in
IE, Chrome or Firefox (typically, key combination `Ctrl`+`Shift`+`I` or `Alt`+`Cmd`+`I`), tab Network,
and reload the bookinfo web page.

You will see and feel that the webpage loads in about 6 seconds:

![Delay]({% image_path testuser-delay.png %})

The reviews section will show: **Sorry, product reviews are currently unavailable for this book**:

####2. Use tracing to identify the bug

---

The reason that the entire reviews service has failed is because our BookInfo application has
a bug. The timeout between the `productpage` and `reviews` service is less (3s times 2 retries == 6s total)
than the timeout between the reviews and ratings service (10s). These kinds of bugs can occur in
typical enterprise applications where different teams develop different microservices independently.

Identifying this timeout mismatch is not so easy by observing the application, but is very easy when using
Istio's built-in tracing capabilities. We will explore tracing in depth later on in this scenario and re-visit
this issue.

At this point we would normally fix the problem by either increasing the `productpage` timeout or
decreasing the `reviews` -> `ratings` service timeout, terminate and restart the fixed microservice,
and then confirm that the `productpage` returns its response without any errors.

However, we already have this fix running in `v3` of the reviews service, so we can simply fix the
problem by migrating all traffic to `reviews:v3`. We'll do this in the next step!

####3. Traffic Shifting

---

This step shows you how to gradually migrate traffic from an old to new version of a service.
With Istio, we can migrate the traffic in a gradual fashion by using a sequence of rules with
weights less than 100 to migrate traffic in steps, for example 10, 20, 30, … 100%. For simplicity
this task will migrate the traffic from `reviews:v1` to `reviews:v3` in just two steps: 50%, 100%.

Now that we've identified and fixed the bug, let's undo our previous testing routes. Execute:

~~~shell
oc delete -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml \
           -f samples/bookinfo/kube/route-rule-ratings-test-delay.yaml
~~~

At this point, we are back to sending all traffic to `reviews:v1`. Access the application at 

`http://istio-ingress-istio-system.$ROUTE_SUFFIX/productpage`
and verify that no matter how many times you reload your browser, you'll always get no ratings stars, since
`reviews:v1` doesn't ever access the `ratings` service:

![no stars]({% image_path stars-none.png %})

Open the Grafana dashboard and verify that the ratings service is receiving no traffic at all:

* Grafana Dashboard at 

`http://grafana-istio-system.$ROUTE_SUFFIX/dashboard/db/istio-dashboard`

Scroll down to the `reviews` service and observe that all traffic from `productpage` to to `reviews:v2` and
`reviews:v3` have stopped, and that only `reviews:v1` is receiving requests:

![no traffic]({% image_path ratings-no-traffic.png %})

In Grafana you can click on each service version below each graph to only show one graph at a time. Try it by
clicking on `productpage.istio-system-v1 -> v1 : 200`. This shows a graph of all requests coming from
`productpage` to `reviews` version `v1` that returned HTTP 200 (Success). You can then click on
`productpage.istio-system-v1 -> v2 : 200` to verify no traffic is being sent to `reviews:v2`:

![no traffic 2]({% image_path ratings-no-traffic-v2.png %})

####4. Migrate users to v3

---

To start the process, let's send half (50%) of the users to our new `v3` version with the fix, to do a canary test.
Execute the following command which replaces the `reviews-default` rule with a new rule:

`oc replace -f samples/bookinfo/kube/route-rule-reviews-50-v3.yaml`

Inspect the new rule:

`oc get routerule reviews-default -o yaml`

Notice the new `weight` elements:

~~~yaml
  route:
  - labels:
      version: v1
    weight: 50
  - labels:
      version: v3
    weight: 50
~~~

Open the Grafana dashboard and verify this:

* Grafana Dashboard at 

`http://grafana-istio-system.$ROUTE_SUFFIX/dashboard/db/istio-dashboard`

Scroll down to the `reviews` service and observe that half the traffic goes to each of `v1` and `v3` and none goes
to `v2`:

![half traffic]({% image_path reviews-v1-v3-half.png %})

At this point, we see some traffic going to `v3` and are happy with the result. Access the application at 

`http://istio-ingress-istio-system.$ROUTE_SUFFIX/productpage`
and verify that you either get no ratings stars (`v1`) or _red_ ratings stars (`v3`).

We are now happy with the new version `v3` and want to migrate everyone to it. Execute:

`oc replace -f samples/bookinfo/kube/route-rule-reviews-v3.yaml`

Once again, open the Grafana dashboard and verify this:

* Grafana Dashboard at 

`http://grafana-istio-system.$ROUTE_SUFFIX/dashboard/db/istio-dashboard`

Scroll down to the `reviews` service and observe that all traffic is now going to `v3`:

![all v3 traffic]({% image_path reviews-v3-all.png %})

Also, Access the application at 

`http://istio-ingress-istio-system.$ROUTE_SUFFIX/productpage`
and verify that you always get _red_ ratings stars (`v3`).

**Congratulations!** In this task we migrated traffic from an old to new version of the reviews service using Istio’s
weighted routing feature. Note that this is very different than version migration using deployment
features of OpenShift, which use instance scaling to manage the traffic. With Istio, we can allow
the two versions of the reviews service to scale up and down independently, without affecting the
traffic distribution between them. For more about version routing with autoscaling, check out
[Canary Deployments using Istio](https://istio.io/blog/canary-deployments-using-istio.html).

In the next step, we will explore circuit breaking, which is useful for avoiding cascading failures
and overloaded microservices, giving the system a chance to recover and minimize downtime.

####5. Enable Circuit Breaker

---

In this step, you will configure an Istio Circuit Breaker to protect the calls from `reviews` to `ratings` service.
If the `ratings` service gets overloaded due to call volume, Istio (in conjunction with Kubernetes) will limit
future calls to the service instances to allow them to recover.

Circuit breaking is a critical component of distributed systems.
It’s nearly always better to fail quickly and apply back pressure downstream
as soon as possible. Istio enforces circuit breaking limits at the network
level as opposed to having to configure and code each application independently.

Istio supports various types of circuit breaking:

* **Cluster maximum connections**: The maximum number of connections that Istio will establish to all hosts in a cluster.
* **Cluster maximum pending requests**: The maximum number of requests that will be queued while waiting for a
ready connection pool connection.
* **Cluster maximum requests**: The maximum number of requests that can be outstanding to all hosts in a
cluster at any given time. In practice this is applicable to HTTP/2 clusters since HTTP/1.1 clusters are
governed by the maximum connections circuit breaker.
* **Cluster maximum active retries**: The maximum number of retries that can be outstanding to all hosts
in a cluster at any given time. In general Istio recommends aggressively circuit breaking retries so that
retries for sporadic failures are allowed but the overall retry volume cannot explode and cause large
scale cascading failure.

> Note that HTTP 2 uses a single connection and never queues (always multiplexes), so max connections and
max pending requests are not applicable.

Each circuit breaking limit is configurable and tracked on a per upstream cluster and per priority basis.
This allows different components of the distributed system to be tuned independently and have different limits.
See the [Istio Circuit Breaker Spec](https://istio.io/docs/reference/config/traffic-rules/destination-policies.html#istio.proxy.v1.config.CircuitBreaker) for more details.

Let's add a circuit breaker to the calls to the `ratings` service. Instead of using a _RouteRule_ object,
circuit breakers in isto are defined as _DestinationPolicy_ objects. DestinationPolicy defines client/caller-side policies
that determine how to handle traffic bound to a particular destination service. The policy specifies
configuration for load balancing and circuit breakers.

Add a circuit breaker to protect calls destined for the `ratings` service:

~~~shell
oc create -f - <<EOF
    apiVersion: config.istio.io/v1alpha2
    kind: DestinationPolicy
    metadata:
      name: ratings-cb
    spec:
      destination:
        name: ratings
        labels:
          version: v1
      circuitBreaker:
        simpleCb:
          maxConnections: 1
          httpMaxPendingRequests: 1
          httpConsecutiveErrors: 1
          sleepWindow: 15m
          httpDetectionInterval: 10s
          httpMaxEjectionPercent: 100
EOF
~~~

We set the `ratings` service's maximum connections to 1 and maximum pending requests to 1. Thus, if we send more
than 2 requests within a short period of time to the reviews service, 1 will go through, 1 will be pending,
and any additional requests will be denied until the pending request is processed. Furthermore, it will detect any hosts that
return a server error (5XX) and eject the pod out of the load balancing pool for 15 minutes. You can visit
here to check the
[Istio spec](https://istio.io/docs/reference/config/traffic-rules/destination-policies.html#istio.proxy.v1.config.CircuitBreaker.SimpleCircuitBreakerPolicy)
for more details on what each configuration parameter does.

####6. Overload the service

---

Let's use some simple `curl` commands to send multiple concurrent requests to our application, and witness the
circuit breaker kicking in opening the circuit.

Execute this to simulate a number of users attampting to access the application simultaneously:


~~~shell
    for i in {1..10} ; do
        curl 'http://istio-ingress-istio-system.[[HOST_SUBDOMAIN]]-80-[[KATACODA_HOST]].environments.katacoda.com/productpage?foo=[1-1000]' >& /dev/null &
    done
~~~

Due to the very conservative circuit breaker, many of these calls will fail with HTTP 503 (Server Unavailable). To see this,
open the Grafana console:

* Grafana Dashboard at 

`http://grafana-istio-system.$ROUTE_SUFFIX/dashboard/db/istio-dashboard`

> **NOTE**: It make take 10-20 seconds before the evidence of the circuit breaker is visible
within the Grafana dashboard, due to the not-quite-realtime nature of Prometheus metrics and Grafana
refresh periods and general network latency.

Notice at the top, the increase in the number of **5xxs Responses** at the top right of the dashboard:

![5xxs]({% image_path 5xxs.png %})

Below that, in the **Service Mesh** section of the dashboard observe that the services are returning 503 (Service Unavailable) quite a lot:

![5xxs]({% image_path 5xxs-services.png %})

That's the circuit breaker in action, limiting the number of requests to the service. In practice your limits would be much higher

####7. Stop overloading

---

Before moving on, stop the traffic generator by clicking here to stop them:

`for i in {1..10} ; do kill %${i} ; done`

####8. Pod Ejection

---

In addition to limiting the traffic, Istio can also forcibly eject pods out of service if they are running slowly
or not at all. To see this, let's deploy a pod that doesn't work (has a bug).

First, let's define a new circuit breaker, very similar to the previous one but without the arbitrary connection
limits. To do this, execute:

~~~shell
oc replace -f - <<EOF
    apiVersion: config.istio.io/v1alpha2
    kind: DestinationPolicy
    metadata:
      name: ratings-cb
    spec:
      destination:
        name: ratings
        labels:
          version: v1
      circuitBreaker:
        simpleCb:
          httpConsecutiveErrors: 1
          sleepWindow: 15m
          httpDetectionInterval: 10s
          httpMaxEjectionPercent: 100
EOF
~~~

This policy says that if any instance of the `ratings` service fails more than once, it will be ejected for
15 minutes.

Next, deploy a new instance of the `ratings` service which has been misconfigured and will return a failure
(HTTP 500) value for any request. Execute:

`${ISTIO_HOME}/bin/istioctl kube-inject -f ~/projects/ratings/broken.yaml | oc create -f -`

Verify that the broken pod has been added to the `ratings` load balancing service:

`oc get pods -l app=ratings`

You should see 2 pods, including the broken one:

~~~shell
NAME                                 READY     STATUS    RESTARTS   AGE
ratings-v1-3080059732-5ts95          2/2       Running   0          3h
ratings-v1-broken-1694306571-c6zlk   2/2       Running   0          7s
~~~

Save the name of this pod to an environment variable:

`BROKEN_POD_NAME=$(oc get pods -l app=ratings,broken=true -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}')`

Requests to the `ratings` service will be load-balanced across these two pods. The circuit breaker will
detect the failures in the broken pod and eject it from the load balancing pool for a minimum of 15 minutes.
In the real world this would give the failing pod a chance to recover or be killed and replaced. For mis-configured
pods that will never recover, this means that the pod will very rarely be accessed (once every 15 minutes,
and this would also be very noticeable in production environment monitoring like those we are
using in this workshop).

To trigger this, simply access the application:

* Application Link at 

`http://istio-ingress-istio-system.$ROUTE_SUFFIX/productpage`

Reload the webpage 5-10 times (click the reload icon, or press `CMD-R`, or `CTRL-R`) and notice that you
only see a failure (no stars) ONE time, due to the
circuit breaker's policy for `httpConsecutiveErrors=1`.  After the first error, the pod is ejected from
the load balancing pool for 15 minutes and you should see red stars from now on.

Verify that the broken pod only received one request that failed:

`oc logs -c ratings $BROKEN_POD_NAME`

You should see:

~~~shell
Server listening on: http://0.0.0.0:9080
GET /ratings/0
~~~

You should see one and only one `GET` request, no matter how many times you reload the webpage.
This indicates that the pod has been ejected from the load balancing pool and will not be accessed
for 15 minutes. You can also see this in the Prometheus logs for the Istio Mixer. Open the Prometheus query
console:

* Prometheus UI at 

`http://prometheus-istio-system.$ROUTE_SUFFIX`

In the “Expression” input box at the top of the web page, enter the text: `envoy_cluster_out_ratings_istio_system_svc_cluster_local_http_version_v1_outlier_detection_ejections_active` and click
**Execute**. This expression refers to the number of _active ejections_ of pods from the `ratings:v1` destination that have failed more than the value of the `httpConsecutiveErrors` which
we have set to 1 (one).

Then, click the Execute button.

You should see a result of `1`:

![5xxs]({% image_path prom-outlier.png %})

In practice this means that the failing pod will not receive any traffic for the timeout period, giving it a chance
to recover and not affect the user experience.

**Congratulations!** Circuit breaking is a critical component of distributed systems. When we apply a circuit breaker to an
entity, and if failures reach a certain threshold, subsequent calls to that entity should automatically
fail without applying additional pressure on the failed entity and paying for communication costs.

In this step, you implemented the Circuit Breaker microservice pattern without changing any of the application code.
This is one additional way to build resilient applications, ones designed to deal with failure rather than go to great lengths
to avoid it.

In the next step, we will explore rate limiting, which can be useful to give different service levels to
different customers based on policy and contractual requirements

> **NOTE**: Before moving on, in case your simulated user loads are still running, kill them with:

`for i in {1..10} ; do kill %${i}; done`

##### More references

* [Istio Documentation](https://istio.io/docs)
* [Christian Posta's Blog on Envoy and Circuit Breaking](http://blog.christianposta.com/microservices/01-microservices-patterns-with-envoy-proxy-part-i-circuit-breaking/)


####9. Rate Limiting

---

In this step, we will use Istio's Quota Management feature to apply
a rate limit on the `ratings` service.

* Quotas in Istio
Quota Management enables services to allocate and free quota on a
based on rules called _dimensions_. Quotas are used as a relatively
simple resource management tool to provide some fairness between
service consumers when contending for limited resources.
Rate limits are examples of quotas, and are handled by the
[Istio Mixer](https://istio.io/docs/concepts/policy-and-control/mixer.html).

* Generate some traffic

As before, let's start up some processes to generate load on the app. Execute this command:

~~~shell
while true; do
    curl -o /dev/null -s -w "%{http_code}\n" \
      http://istio-ingress-istio-system.[[HOST_SUBDOMAIN]]-80-[[KATACODA_HOST]].environments.katacoda.com/productpage
  sleep .2
done
~~~

This command will endlessly access the application and report the HTTP status result in a separate terminal window.

With this application load running, we can witness rate limits in action.

####10. Add a rate limit

---

Execute the following command:

`oc create -f samples/bookinfo/kube/mixer-rule-ratings-ratelimit.yaml`

This configuration specifies a default 1 qps (query per second) rate limit. Traffic reaching
the `ratings` service is subject to a 1qps rate limit. Verify this with Grafana:

* Grafana Dashboard at 

`http://grafana-istio-system.$ROUTE_SUFFIX/dashboard/db/istio-dashboard`

Scroll down to the `ratings` service and observe that you are seeing that some of the requests sent
from `reviews:v3` service to the `ratings` service are returning HTTP Code 429 (Too Many Requests).

![5xxs]({% image_path ratings-overload.png %})

In addition, at the top of the dashboard, the '4xxs' report shows an increase in 4xx HTTP codes. We are being
rate-limited to 1 query per second:

![5xxs]({% image_path ratings-4xxs.png %})

####11. Inspect the rule

---

Take a look at the new rule:

`oc get memquota handler -o yaml`

In particular, notice the _dimension_ that causes the rate limit to be applied:

~~~yaml
# The following override applies to 'ratings' when
# the source is 'reviews'.
- dimensions:
    destination: ratings
    source: reviews
  maxAmount: 1
  validDuration: 1s
~~~

You can also conditionally rate limit based on other dimensions, such as:

* Source and Destination project names (e.g. to limit developer projects from overloading the production services during testing)
* Login names (e.g. to limit certain customers or classes of customers)
* Source/Destination hostnames, IP addresses, DNS domains, HTTP Request header values, protocols
* API paths
* [Several other attributes](https://istio.io/docs/reference/config/mixer/attribute-vocabulary.html)

####12. Remove the rate limit

---

Before moving on, execute the following to remove our rate limit:

`oc delete -f samples/bookinfo/kube/mixer-rule-ratings-ratelimit.yaml`

Verify that the rate limit is no longer in effect. Open the dashboard:

* Grafana Dashboard at 

`http://grafana-istio-system.$ROUTE_SUFFIX/dashboard/db/istio-dashboard`

Notice at the top that the `4xx`s dropped back down to zero.

![5xxs]({% image_path ratings-4xxs-gone.png %})

**Congratulations!** In the final step, we'll explore distributed tracing and how it can help diagnose and fix issues in
complex microservices architectures. Let's go!


####13. Tracing

---

This step shows you how Istio-enabled applications automatically collect
_trace spans_ telemetry and can visualize it with tools like using Jaeger or Zipkin.
After completing this task, you should
understand all of the assumptions about your application and how to have it
participate in tracing, regardless of what language/framework/platform you
use to build your application.

##### Tracing Goals

Developers and engineering organizations are trading in old, monolithic systems
for modern microservice architectures, and they do so for numerous compelling
reasons: system components scale independently, dev teams stay small and agile,
deployments are continuous and decoupled, and so on.

Once a production system contends with real concurrency or splits into many
services, crucial (and formerly easy) tasks become difficult: user-facing
latency optimization, root-cause analysis of backend errors, communication
about distinct pieces of a now-distributed system, etc.

##### What is a trace?

At the highest level, a trace tells the story of a transaction or workflow as
it propagates through a (potentially distributed) system. A trace is a directed
acyclic graph (DAG) of _spans_: named, timed operations representing a
contiguous segment of work in that trace.

Each component (microservice) in a distributed trace will contribute its
own span or spans. For example:

![Spans]({% image_path tracing.png %})

This type of visualization adds the context of time, the hierarchy of
the services involved, and the serial or parallel nature of the process/task
execution. This view helps to highlight the system's critical path. By focusing
on the critical path, attention can focus on the area of code where the most
valuable improvements can be made. For example, you might want to trace the
resource allocation spans inside an API request down to the underlying blocking calls.

####14. Access Jaeger Console

---

With our application up and our script running to generate loads, visit the Jaeger Console:

* Jaeger Query Dashboard at 

`http://jaeger-query-istio-system.$ROUTE_SUFFIX`

![jager console]({% image_path jag-console.png %})

Select `istio-ingress` from the _Service_ dropdown menu, change the value of **Limit Results** to `200` and click **Find Traces**:

![jager console]({% image_path jag-console2.png %})

In the top right corner, a duration vs. time scatter plot gives a visual representation of the results, showing how and when
each service was accessed, with drill-down capability. The bottom right includes a list of all spans that were traced over the last
hour (limited to 200).

If you click on the first trace in the listing, you should see the details corresponding
to a recent access to `/productpage`. The page should look something like this:

![jager listing]({% image_path jag-listing.png %})

As you can see, the trace is comprised of _spans_, where each span corresponds to a
microservice invoked during the execution of a `/productpage` request.

The first line represents the external call to the entry point of our application controlled by
 `istio-ingress`. It in thrn calls the `productpage` service. Each line below
represents the internal calls to the other services to construct the result, including the
time it took for each service to respond.

To demonstrate the value of tracing, let's re-visit our earlier timeout bug! If you recall, we had
injected a 7 second delay in the `ratings` microservice for our user _jason_. So when we loaded the
web page it should have taken 7 seconds before showing the star ratings.

In reality, the webpage loaded in 6 seconds and we saw no rating stars! Why did this happen? We know
from earlier that it was because the timeout from `reviews`->`ratings` was much shorter than the `ratings`
timeout itself, so it prematurely failed the access to `ratings` after 2 retries (of 3 seconds each), resulting
in a failed webpage after 6 seconds. But can we see this in the tracing? Yes, we can!

To see this bug, open the Jaeger tracing console:

* Jaeger Query Dashboard at 

`http://jaeger-query-istio-system.$ROUTE_SUFFIX`

Since users of our application were reporting lengthy waits of 5 seconds or more, let's look for traces
that took at least 5 seconds. Select these options for the query:

* **Service**: `istio-ingress`
* **Min Duration**: `5s`

Then click **Find Traces**. Change the sorting to **Longest First** to see the ones that took the longest.
The result list should show several spans with errors:

![jager listing]({% image_path jag-last10.png %})

Click on the top-most span that took ~10s and open details for it:

![jager listing]({% image_path jag-2x3.png %})

Here you can see the `reviews` service takes 2 attempts to access the `ratings` service, with each attempt
timing out after 3 seconds. After the second attempt, it gives up and returns a failure back to the product
page. Meanwhile, each of the attempts to get ratings finally succeeds after its fault-injected 7 second delay,
but it's too late as the reviews service has already given up by that point.

The timeouts are incompatible, and need to be adjusted. This is left as an exercise to the reader.

Istio’s fault injection rules and tracing capabilities help you identify such anomalies without impacting end users.

> **NOTE**: Let's stop the load generator running against our app. Navigate to **Terminal 2** and type
`CTRL-C` to stop the generator or click `clear`.

**Congratulations!** Distributed tracing speeds up troubleshooting by allowing developers to quickly understand
how different services contribute to the overall end-user perceived latency. In addition,
it can be a valuable tool to diagnose and troubleshoot distributed applications.

####15. Enable RH-SSO

---

#### Summary

In this scenario you used Istio to implement many of the
Istio provides an easy way to create a network of deployed services with load balancing, service-to-service authentication, monitoring, and more, without requiring any changes in service code. You add Istio support to services by deploying a special sidecar proxy throughout your environment that intercepts all network communication between microservices, configured and managed using Istio’s control plane functionality.

Technologies like containers and container orchestration platforms like OpenShift solve the deployment of our distributed
applications quite well, but are still catching up to addressing the service communication necessary to fully take advantage
of distributed microservice applications. With Istio you can solve many of these issues outside of your business logic,
freeing you as a developer from concerns that belong in the infrastructure. Congratulations!

Additional Resources:

* [Istio on OpenShift via Veer Muchandi](https://github.com/VeerMuchandi/istio-on-openshift)
* [Envoy resilience examples](http://blog.christianposta.com/microservices/00-microservices-patterns-with-envoy-proxy-series/)
* [Istio and Kubernetes workshop from KubeCon 2017 via Zach Butcher, et. al.]()
* [Istio and Kubernetes workshop](https://github.com/retroryan/istio-workshop)
* [Bookinfo from http://istio.io](https://istio.io/docs/tasks/traffic-management/request-routing.html)