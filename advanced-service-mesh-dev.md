## Lab3 - Advanced Service Mesh Development

In this lab, you will use advanced service mesh features such as **Fault Injection**, **Traffic Shifting**, **Circuit Breaking**, and **Rate Limiting** with a different application - the Coolstore microservice developed in prior labs (i.e Catalog, Inventory) that you developed and deployed to OpenShift cluster in _Module 1_ or/and _Module 2_.

##### If this is the first module you are doing today

If you have already deployed the _inventory_ and _catalog_ microservices from Module 1, you can skip this step!

If you haven't done Module 1 or Module 2 today, **or you didn't quite complete them**, you should deploy the coolstore application and microservices by executing the following shell scripts in CodeReady Workspaces Terminal:

> Replace `userXX` with your username before running these commands:

`sh /projects/cloud-native-workshop-v2m3-labs/istio/scripts/deploy-inventory.sh userXX`

`sh /projects/cloud-native-workshop-v2m3-labs/istio/scripts/deploy-catalog.sh userXX`

This will build and deploy the inventory and catalog components into the `userXX-inventory` and `userXX-catalog` projects in OpenShift. They won't automatically get Istio sidecar proxy containers yet, but you'll add that in the next step!

####1. Configuring Automatic Sidecar Injection in Coolstore Microservices

Let's go to the [Kiali console](https://kiali-istio-system.{{ROUTE_SUBDOMAIN}}/){:target="_blank"} once again to confirm that the microservices (_Catalog_, _Inventory_) are not running with a _sidecar_ and are not yet visible in the service mesh.

Click on **Applications** on the left menu. Then, using the `Namespace` drop-down at the top, type in `userXX` (your username) to filter the list, and check the `userXX-catalog` and `userXX-inventory` namespaces, and de-select the `userXX-bookinfo` namespace. You will see **Missing Sidecar** in 4 applications.

![istio]({% image_path kiali_missing_sidecar.png %})

Upstream Istio community installations rely on the existence of a **proxy sidecar** within the application’s pod to provide service mesh capabilities to the application. You can include the proxy sidecar by using a manual process before deployment. However, automatic injection ensures that your application contains the appropriate configuration for your service mesh at the time of its deployment.

_Automatic injection of the sidecar_ is supported by using the _sidecar.istio.io/inject_ annotation within your application yaml file. You will add this in the next step.

> Upstream Istio community installations require a specific label on the namespace after which all pods in that namespace are injected with the sidecar.  The OpenShift Service Mesh approach requires you to opt in to injection using an annotation with no need to label namspaces. This method requires fewer privileges and does not conflict with other OpenShift capabilities such as builder pods.

Back in the the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"}, select the _userXX-inventory_ project in the project drop-down, then go to **Workloads > Deployment Configs** on the left menu, and click on _inventory-database_ and then click the _YAML_ tab.

![istio]({% image_path inventory_db_dc.png %})

Add the following annotation in `spec.template.metadata.annotations` path in the YAML and click on **Save**.

`sidecar.istio.io/inject: "true"`

> **NOTE** Be sure to place this annotation in the correct `annotations` block under _spec > template > metadata > annotations_ and match the same indentation as the annotation immediately above!

![istio]({% image_path inventory_db_inject_sidecar.png %})

You will see a new **istio-proxy** container and _inventory-database_ container in the "Pod Details" page when you navigate to _Workloads > Pods > inventory-database-xxxxx_. This new container will intercept traffic to and from the application pod and potentially alter and/or re-route it depending on service mesh configuration.

![istio]({% image_path inventory_db_sidecar.png %})

Now you will inject a sidecar container to application container (Inventory) as well. First, we need to remove the healhcheck of the inventory service as it will also be proxied and as we have no route for it, will be rejected (and the pod killed since Kubernetes cannot access the health check!). Alternative health checks involve running commands directly in the container but we'll just remove it for now. Remove it with:

`oc set probe dc/inventory-quarkus --remove --readiness --liveness -n userXX-inventory`

> Replace `userXX` with your username!

Ensure the new deployemnt is successfully rolled out:

`oc rollout status -w dc/inventory-quarkus -n userXX-inventory`

Navigate _Workloads > Deployment Configs_ on the left menu. Select _userXX-inventory_ project and click on _inventory-quarkus_ and then the `YAML` tab.

![istio]({% image_path inventory_dc.png %})

We'll add the same annotation as before:

`sidecar.istio.io/inject: "true"`

> **NOTE** Be sure to place this annotation in the correct `annotations` block under _spec > template > metadata > annotations_ and match the same indentation as the annotation immediately above!

![istio]({% image_path inventory_inject_sidecar.png %})

Again you will see **istio-proxy** container and _inventory-quarkus_ container in the "Pod Details" page when you navigate _Workloads > Pods > inventory-quarkus-xxxxx_:

![istio]({% image_path inventory_sidecar.png %})

Next, do the same for the catalog and catalog's database.  Go to **Workloads > Deployment Configs* on the left menu, select _userXX-catalog_ project and click on _catalog-database_.

![istio]({% image_path catalog_db_dc.png %})

Then click on **YAML** tab and add the following annotation in `spec.template.metadata.annotations` path and click on **Save**.

`sidecar.istio.io/inject: "true"`

![istio]({% image_path catalog_db_inject_sidecar.png %})

You will see **istio-proxy** container and _catalog-database_ container in Pod Details page when you navigate _Workloads > Pods > catalog-database-xxxxx_.

![istio]({% image_path catalog_db_sidecar.png %})

Next, let's inject a sidecar container to application container (Catalog) as well,
go to **Workloads > Deployment Configs* on the left menu, select _userXX-catalog_ project and click on _catalog-springboot_.

![istio]({% image_path catalog_dc.png %})

Add the same annotation (on the YAML tab):

`sidecar.istio.io/inject: "true"`

![istio]({% image_path catalog_inject_sidecar.png %})

You will see **istio-proxy** container and _catalog-springboot_ container in the "Pod Details" page when you navigate _Workloads > Pods > catalog-springboot-xxxxx_:

![istio]({% image_path catalog_sidecar.png %})

Let's make sure if inventory and catalog services are working correctly via accessing _Catalog Route URL_ in your browser. You can also find the URL via _Networking > Routes_ in OpenShift web console, after selecting the `userXX-catalog` from the _namespace_ dropdown menu. Open the URL in your browser:

* Catalog UI (replace `userXX` with your username): http://catalog-springboot-userXX-catalog.{{ROUTE_SUBDOMAIN}}

You will see the following web page including **Inventory Quantity** if the catalog service can access the inventory service via _Istio proxy sidecar_:

![istio]({% image_path catalog_route_sidecar.png %})

> Leave this page open as the _Catalog UI browser_ creates traffic (every 2 seconds) between services, which is useful for testing.

Now, reload **Applications** in [Kiali console](https://kiali-istio-system.{{ROUTE_SUBDOMAIN}}/){:target="_blank"} and verify that the _Missing sidecar_ warning is no longer present:

![istio]({% image_path kiali_injecting_sidecar.png %})

Also, go to the Service Graph page and check _userXX-inventory_, _userXX-catalog_ in Namespace, check **Traffic Animation** in _Display_ for understanding
the traffic flow from catalog service to inventory service:

![istio]({% image_path kiali_graph_sidecar.png %})

####2. Fault Injection

---

This step will walk you through how to use **Fault Injection** to test the end-to-end failure recovery capability of the application as a whole. An incorrect configuration of the failure recovery policies could result in unavailability of critical services. Examples of incorrect configurations include incompatible or restrictive timeouts across service calls.

_Istio_ provides a set of failure recovery features that can be taken advantage of by the services
in an application. Features include:

* Timeouts
* Bounded retries with timeout budgets and variable jitter between retries
* Limits on number of concurrent connections and requests to upstream services
* Active (periodic) health checks on each member of the load balancing pool
* Fine-grained circuit breakers (passive health checks) – applied per instance in the load balancing pool

These features can be dynamically configured at runtime through Istio’s traffic management rules.

A combination of active and passive health checks minimizes the chances of accessing an unhealthy service. When combined with platform-level health checks (such as readiness/liveness probes in OpenShift), applications can ensure that unhealthy pods/containers/VMs can be quickly weeded out of the service mesh, minimizing the
request failures and impact on latency.

Together, these features enable the service mesh to tolerate failing nodes and prevent localized failures from cascading instability to other nodes.

Istio enables protocol-specific _fault injection_ into the network (instead of killing pods) by delaying or corrupting packets at TCP layer.

Two types of faults can be injected:

 * _Delays_ are timing failures. They mimic increased network latency or an overloaded upstream service.
 * _Aborts_ are crash failures. They mimic failures in upstream services. Aborts usually manifest in the form of HTTP error codes or TCP connection failures.


##### Inject a fault

To test our application microservices for resiliency, we will inject a failure in **50%** of the requests to the _inventory_ service, causing the service to appear to fail (and return `HTTP 5xx` errors).

First, add the following label in the Inventory service to use a _virtual service_. In the OpenShift Web Consle, select the _userXX-inventory_ project in the project selector drop-down, then navigate to _Networking > Services_ in the left menu, and select _inventory-quarkus_.

![fault-injection]({% image_path inventory_svc_.png %})

Click on **YAML** tab and add the following variables at the _metadata > labels_ area of the YAML file as shown:

`service: inventory-quarkus`

![fault-injection]({% image_path inventory_svc_add_label.png %})

Click on **Save**.

In CodeReady, open the empty **inventory-default.yaml** file in the `/projects/cloud-native-workshop-v2m3-labs/inventory/rules/ `directory. Add the below code to the file to create a gateway and virtual service:

> You'll need to replace `YOUR_INVENTORY_GATEWAY_URL` with the route URL for the inventory service, which looks like `inventory-quarkus-userXX-inventory.{{ROUTE_SUBDOMAIN}}` (replace `userXX` with your username). There are two places to make this substitution, so do them both!

~~~yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: inventory-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - 'YOUR_INVENTORY_GATEWAY_URL'
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: inventory-default
spec:
  hosts:
  - 'YOUR_INVENTORY_GATEWAY_URL'
  gateways:
  - inventory-gateway
  http:
    - match:
        - uri:
            exact: /services/inventory
        - uri:
            exact: /
      route:
        - destination:
            host: inventory-quarkus
            port:
              number: 8080
~~~

![fault-injection]({% image_path inventory-default-gateway.png %})

Delete the old direct route that was setup earlier with:

`oc delete route/inventory-quarkus -n userXX-inventory`

Create the new Istio-powered route by running the following command via CodeReady Workspaces Terminal to create this object in OpenShift:

`oc create -f /projects/cloud-native-workshop-v2m3-labs/inventory/rules/inventory-default.yaml -n userXX-inventory`

Now, you can test if the inventory service works correctly via accessing the **YOUR_INVENTORY_GATEWAY_URL** in your browser:

`i.e. http://inventory-quarkus-userXX-inventory.{{ ROUTE_SUBDOMAIN }}` (replace `userXX` with your username)

![fault-injection]({% image_path inventory-ui-gateway.png %})

Let's inject a failure (_500 status_) in **50%** of requests to _inventory_ microservices. Edit _inventory-default.yaml_ as below.

Open **inventory-vs-fault.yaml** file in `/projects/cloud-native-workshop-v2m3-labs/inventory/rules/` and copy the following codes.

> You need to replace all `YOUR_INVENTORY_GATEWAY_URL` with the previous route URL that you copied earlier.

~~~yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: inventory-fault
spec:
  hosts:
  - 'YOUR_INVENTORY_GATEWAY_URL'
  gateways:
  - inventory-gateway
  http:
    - fault:
         abort:
           httpStatus: 500
           percentage:
             value: 50
      route:
        - destination:
            host: inventory-quarkus
            port:
              number: 8080
~~~

![fault-injection]({% image_path inventory-vs-error.png %})

Before creating a new **inventory-fault VirtualService**, we need to delete the existing inventory-default virtualService. Run the following command via CodeReady Workspaces Terminal:

`oc delete virtualservice/inventory-default -n userXX-inventory` (replace `userXX` with your username)

Then create a new virtualservice and gateway with this command:

`oc create -f /projects/cloud-native-workshop-v2m3-labs/inventory/rules/inventory-vs-fault.yaml -n userXX-inventory`

Let's find out if the fault injection works corectly via accessing the Inventory gateway once again. You will see that the **Status** of CoolStore Inventory continues to change between **DEAD** and **OK**:

![fault-injection]({% image_path inventory-dead-ok.png %})

In the **Kiali** console you will also see failures for 50% of traffic bound for the `inventory `service. You will see `red` traffic from _istio-ingressgateway_ as well as around 50% of requests are displayed as _5xx_ on the right side, _HTTP Traffic_. It may not be *exactly* 50% since some traffic is coming from the catalog and ingress gateway at the same time, but it will approach 50% over time.

![fault-injection]({% image_path inventlry-vs-error-kiali.png %})

Let's now add a 5 second delay for the `inventory` service.

Open **inventory-vs-fault-delay.yaml** file in `/projects/cloud-native-workshop-v2m3-labs/inventory/rules/` and copy the following code into it:

> Again, you need to replace all **YOUR_INVENTORY_GATEWAY_URL** with the previous route URL that you copied earlier.

~~~yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: inventory-fault-delay
spec:
  hosts:
  - 'YOUR_INVENTORY_GATEWAY_URL'
  gateways:
  - inventory-gateway
  http:
    - fault:
         delay:
           fixedDelay: 5s
           percentage:
             value: 100
      route:
        - destination:
            host: inventory-quarkus
            port:
              number: 8080
~~~

![fault-injection]({% image_path inventory-vs-delay.png %})

Before creating a new **inventory-fault-delay VirtualService**, we need to delete the existing inventory-fault VirtualService. Run the following command via CodeReady Workspaces Terminal:

`oc delete virtualservice/inventory-fault -n userXX-inventory`

Then create a new virtualservice and gateway.

`oc create -f /projects/cloud-native-workshop-v2m3-labs/inventory/rules/inventory-vs-fault-delay.yaml -n userXX-inventory`

Go to the **Kiali Graph** you opened earlier and you will see that the `green` traffic from _istio-ingressgateway_ is delayed for requests coming from catalog service. Note that you need to check **Traffic Animation** in the _Display_ select box.

![fault-injection]({% image_path inventlry-vs-delay-kiali.png %})

If the Inventory’s front page was set to correctly handle delays, we expect it to load within
approximately 5 seconds. To see the web page response times, open the Developer Tools menu in
IE, Chrome or Firefox (typically, key combination **Ctrl**+**Shift**+**I** or **Alt**+**Cmd**+**I**), select the `Network` tab, and reload the bookinfo web page.

You will see and feel that the webpage loads in about 5 seconds:

![Delay]({% image_path inventory-webui-delay.png %})

Before we will move to the next step, clean up the fault injection and set the default virtual service once again using these commands in a Terminal:

> Don't forget to replace `userXX` with your username!

`oc delete virtualservice/inventory-fault-delay -n userXX-inventory`

`oc delete gateway/inventory-gateway -n userXX-inventory`

`oc create -f /projects/cloud-native-workshop-v2m3-labs/inventory/rules/inventory-default.yaml -n userXX-inventory`

Also, close the tabs in your browser for the Inventory and Catalog services to avoid unnecessary load, and stop the endless `for` loop you started in the beginning of this lab in CodeReady by closing the Terminal window that was running it.

####3. Enable Circuit Breaker

---

In this step, you will configure a circuit Breaker to protect the calls to `Inventory` service.
If the `Inventory` service gets overloaded due to call volume, Istio will limit
future calls to the service instances to allow them to recover.

Circuit breaking is a critical component of distributed systems.
It’s nearly always better to fail quickly and apply back pressure upstream
as soon as possible. Istio enforces circuit breaking limits at the network
level as opposed to having to configure and code each application independently.

Istio supports various types of conditions that would trigger a circuit break:

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

> Note that **HTTP2** uses a single connection and never queues (always multiplexes), so max connections and max pending requests are not applicable.

Each circuit breaking limit is configurable and tracked on a per upstream cluster and per priority basis. This allows different components of the distributed system to be tuned independently and have different limits. See the [Envoy’s circuit breaker](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking){:target="_blank"} for more details.

Let's add a circuit breaker to the calls to the **Inventory service**. Instead of using a _VirtualService_ object, circuit breakers in Istio are defined as _DestinationRule_ objects. DestinationRule defines policies that apply to traffic intended for a service after routing has occurred. These rules specify configuration for load balancing, connection pool size from the sidecar, and outlier detection settings to detect and evict unhealthy hosts from the load balancing pool.

Open the empty **inventory-cb.yaml** file in `/projects/cloud-native-workshop-v2m3-labs/inventory/rules/` and add this code to the file to enable circuit breaking when calling the Inventory service:

~~~yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: inventory-cb
spec:
  host: inventory-quarkus
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
~~~

![circuit-breaker]({% image_path inventory-circuit-breaker.png %})

Run the following command via CodeReady Workspaces Terminal to then create the rule:

`oc create -f /projects/cloud-native-workshop-v2m3-labs/inventory/rules/inventory-cb.yaml -n userXX-inventory`

We set the Inventory service's maximum connections to 1 and maximum pending requests to 1. Thus, if we send more than 2 requests within a short period of time to the inventory service, 1 will go through, 1 will be pending, and any additional requests will be denied until the pending request is processed. Furthermore, it will detect any hosts that return a server error (HTTP 5xx) and eject the pod out of the load balancing pool for 15 minutes. You can visit here to check the [Istio spec](https://istio.io/docs/reference/config/traffic-rules/destination-policies.html#istio.proxy.v1.config.CircuitBreaker.SimpleCircuitBreakerPolicy){:target="_blank"} for more details on what each configuration parameter does.

####4. Overload the service

---

Let's use simple **curl** commands to send multiple concurrent requests to our application, and witness the circuit breaker kicking in and opening the circuit.

Execute this to simulate a number of users attampting to access the gateway URL simultaneously in CodeReady Workspaces Terminal.

> Replace `YOUR_INVENTORY_GATEWAY_URL` with your custom inventory URL, e.g. `http://inventory-quarkus-userXX-inventory.{{ ROUTE_SUBDOMAIN }}`.

~~~shell
	for i in {1..50} ; do
		curl 'http://YOUR_IVENTORY_GATEWAY_URL/services/inventory' >& /dev/null &
	done
~~~

Due to the very conservative circuit breaker, many of these calls will fail with HTTP 503 (Server Unavailable). To see this,
open the _Istio Service Mesh Dashboard_ in the [Grafana console](https://grafana-istio-system.{{ROUTE_SUBDOMAIN}}/) and select `inventory-quarkus.userXX-inventory.svc.cluster.local` service:

> `NOTE`: It make take 10-20 seconds before the evidence of the circuit breaker is visible within the Grafana dashboard, due to the not-quite-realtime nature of Prometheus metrics and Grafana refresh periods and general network latency.

![circuit-breaker]({% image_path inventory-circuit-breaker-grafana.png %})

That's the circuit breaker in action, limiting the number of requests to the service. In practice your limits would be much higher.

####5. Stop overloading

---

Before moving on, stop the traffic generator by executing the following commands in CodeReady Workspaces Terminal:

`for i in {1..50} ; do kill %${i} ; done`

![circuit-breaker]({% image_path inventory-circuit-breaker-stop.png %})

Delete the circuit breaker of the Inventory service via the following commands. You should replace `userXX` with your namespace:

`oc delete destinationrule/inventory-cb -n userXX-inventory`

####6. Enable Authentication using Single Sign-on

---

In this step, you will learn how to enable authenticating **catalog** microservices with Istio, [JSON Web Token(JWT)](https://en.wikipedia.org/wiki/JSON_Web_Token){:target="_blank"}, and
[Red Hat Single Sign-On](https://access.redhat.com/products/red-hat-single-sign-on) in [Red Hat Runtimes](https://www.redhat.com/en/products/application-runtimes){:target="_blank"}.

First, let's remove the direct route to the catalog service. We want traffic to be managed by the service mesh, and not allow direct traffic. Use the following command in the CodeReady Workspaces Terminal:

`oc delete route/catalog-springboot -n userXX-catalog`

In the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"}, select the `userXX-catalog` project, then navigate to _Networking > Services_ from the left menu, select the `catalog-springboot` service

![sso]({% image_path catalog_svc_vs.png %})

Select the YAML tab and add the following label in the catalog service to use a **virtural service**:

`service: catalog-springboot`

Also, since [Istio requires service names](https://istio.io/docs/setup/additional-setup/requirements/) to be named with specific identifiers, change the name of the `8080-tcp` to be named `http` as shown:

![sso]({% image_path catalog_svc_add_label.png %})

Click on **Save**.

In CodeReady, open the **catalog-default.yaml** file in `/projects/cloud-native-workshop-v2m3-labs/catalog/rules/` to make a gateway and virtual service:

> Replace all **YOUR_CATALOG_GATEWAY_URL** with the catlog route URL which will be `catalog-springboot-userXX-catalog.{{ROUTE_SUBDOMAIN}}` but with `userXX` replaced with your username. Change the code in two places after inserting it into the `catalog-default.yaml` file:

~~~yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: catalog-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - 'YOUR_CATALOG_GATEWAY_URL'
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog-default
spec:
  hosts:
  - 'YOUR_CATALOG_GATEWAY_URL'
  gateways:
  - catalog-gateway
  http:
    - match:
        - uri:
            exact: /services/products
        - uri:
            exact: /services/product
        - uri:
            exact: /
      route:
        - destination:
            host: catalog-springboot
            port:
              number: 8080
~~~

![sso]({% image_path catalog-default-gateway.png %})

Then create this object in OpenShift by running the following command via CodeReady Workspaces Terminal:

`oc create -f /projects/cloud-native-workshop-v2m3-labs/catalog/rules/catalog-default.yaml -n userXX-catalog` (replace `userXX` with your username!)

Now, you can test if the catalog service works correctly by accessing the **YOUR_CATALOG_GATEWAY_URL** without _authentication_ in your browser:

`i.e. http://catalog-springboot-userXX-catalog.{{ROUTE_SUBDOMAIN}}`

![sso]({% image_path catalog-ui-gateway.png %})

Let's deploy **Red Hat Single Sign-On (RH-SSO)** that enables service authentication for traffic in the service mesh.

_Red Hat Single Sign-On (RH-SSO)_ is based on the **Keycloak** project and enables you to secure your web applications by providing Web single sign-on (SSO) capabilities based on popular standards such as **SAML 2.0, OpenID Connect and OAuth 2.0**. The RH-SSO server can act as a SAML or OpenID Connect-based Identity Provider, mediating with your enterprise user directory or 3rd-party SSO provider for identity information and your applications via standards-based tokens. The major features include:

 * **Authentication Server** - Acts as a standalone SAML or OpenID Connect-based Identity Provider.
 * **User Federation** - Certified with LDAP servers and Microsoft Active Directory as sources for user information.
 * **Identity Brokering** - Integrates with 3rd-party Identity Providers including leading social networks as identity source.
 * **REST APIs and Administration GUI** - Specify user federation, role mapping, and client applications with easy-to-use Administration GUI and REST APIs.

We will deploy RH-SSO in Catalog project. Run the following commands in CodeReady Workspaces Terminal:

> Note: You need to replace `userXX` with your username and replace `authuserXX` below with your username plus `auth` prefix. For example, `authuser12` or `authuser2`.

~~~shell
oc -n userXX-catalog new-app ccn-sso72 \
   -p SSO_ADMIN_USERNAME=admin \
   -p SSO_ADMIN_PASSWORD=admin \
   -p SSO_REALM=istio \
   -p SSO_SERVICE_USERNAME=authuserXX \
   -p SSO_SERVICE_PASSWORD=openshift
~~~

Wait for RH-SSO to be deployed using this command:

`oc rollout status -w dc/sso -n userXX-catalog` (replace `userXX` with your username)

Once this finishes (it may take a minute or two), in the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"} navigate to _Networking > Routes_ and you will see the route URL as below (in the `userXX-catalog` project):


![sso]({% image_path rhsso_deployment.png %})

Click on **HTTPS** URL(i.e. `secure-sso-userXX-catalog.{{ROUTE_SUBDOMAIN}}`) to access RH-SSO web console as below:

![sso]({% image_path rhsso_landing_page.png %})

Click on _Administration Console_ to configure **Istio** Ream then input the usename and password that you used earlier:

 * Username or email: **admin**
 * Password: **admin**

![sso]({% image_path rhsso_admin_login.png %})

You will see general information of the _Istio Realm_. Click on **Login** tab and de-select (swich off) _Require SSL_ by setting it to _none_ then click on **Save**.

![sso]({% image_path rhsso_istio_realm.png %})

> Red Hat Single Sign-On generates a self-signed certificate the first time it runs. Please note that self-signed certificates don't work to authenticate by Istio so we will change not to use SSL for testing Istio authentication.

Next, create a new RH-SSO _client_ that is for trusted browser apps and web services in our _Istio_ realm.
Go to **Clients** in the left menu then click on **Create**.

![sso]({% image_path rhsso_clients.png %})

Input **ccn-cli** in _Client ID_ field and click on **Save**.

![sso]({% image_path rhsso_clients_create.png %})

On the next screen, you will see details on the **Settings** tab, the only thing you need to do is to input _Valid Redirect URIs_ that can be used after successful login or logout for clients.

> Replace **YOUR_CATALOG_GATEWAY_URL** with your own ingress gateway URL of the catalog service and please note to add **http://** at the front as well as `/*` at the end of URL.

 * Valid Redirect URIs: `http://catalog-springboot-userXX-catalog.{{ ROUTE_SUBDOMAIN }}/*` (replace `userXX` with your username!)

![sso]({% image_path rhsso_clients_settings.png %})

Don't forget to click **Save**!

Now, let's define a role that will be assigned to your credentials, let’s create a simple role called **ccn_auth**. Go to **Roles** in the left menu then click on _Add Role_.

![sso]({% image_path rhsso_roles.png %})

Input **ccn_auth** in _Role Name_ field and click on **Save**.

![sso]({% image_path rhsso_roles_create.png %})

Next let's update the password policy for our _authuser_.

Go to **Users** menu on the left side menu then click on **View all users**.

![sso]({% image_path rhsso_users.png %})

If you click on the `authuserXX` ID then you will find more information such as Details, Attributes, Credentials, Role Mappings, Groups, Contents, and Sessions. You don't need to update any details in this step.

![sso]({% image_path rhsso_istio_users_details.png %})

Go to **Credentials** tab and input the following variables:

 * New Password: **openshift**
 * Password Confirmation: **openshift**
 * Temporary: **OFF**

Make sure to turn off the “Temporary” flag unless you want the authuserXX to have to change his password the first time they authenticate.

Click on **Reset Password**.

![sso]({% image_path rhsso_users_credentials.png %})

Then click on **Change password** in the popup window.

![sso]({% image_path rhsso_users_change_pwd.png %})

Now proceed to the **Role Mappings** tab and assign the role **ccn_auth** via clicking on _Add selected >_.

![sso]({% image_path rhsso_rolemapping.png %})

You will confirm the ccn_auth role in _Assigned Roles_ box.

![sso]({% image_path rhsso_rolemapping_assigned.png %})

Well done, you have enabled RH-SSO to with a custom realm, user and role!

Turning to back to Istio, let's create a user-facing authentication policy using JSON Web Tokens (JWTs). The format is defined in [RFC 7519](https://tools.ietf.org/html/rfc7519){:target="_blank"}. You can find more details how [OAuth 2.0](https://tools.ietf.org/html/rfc6749){:target="_blank"} and [OIDC 1.0](https://openid.net/connect/){:target="_blank"} work in the overall authentication flow.

In CodeReady, open the blank **ccn-auth-config.yml** file in `/projects/cloud-native-workshop-v2m3-labs/catalog/rules/` to create an authentication policy:

> Replace all **YOUR_SSO_HTTP_ROUTE_URL** with your own HTTP route url of SSO container that you created earlier and also replace **userXX** with your username.

You can also get the route url via executing the following commands in CodeReady Workspaces Terminal:

`oc get route -n userXX-catalog secure-sso --template '{{.spec.host}} '`

Use this value to replace `YOUR_SSO_HTTP_ROUTE_URL`. You will also use this later!

~~~yaml
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:
  name: auth-policy
  namespace: userXX-catalog
spec:
  targets:
  - name: catalog-springboot
  origins:
  - jwt:
      issuer: http://YOUR_SSO_HTTP_ROUTE_URL/auth/realms/istio
      jwks_uri: http://YOUR_SSO_HTTP_ROUTE_URL/auth/realms/istio/protocol/openid-connect/certs
  principalBinding: USE_ORIGIN
~~~

The following fields are used above to create a Policy in Istio and are described here:

 * **issuer** - Identifies the issuer that issued the JWT. See [issuer](https://tools.ietf.org/html/rfc7519#section-4.1.1){:target="_blank"} usually a URL or an email address.
 * **jwksUri** - URL of the provider’s public key set to validate signature of the JWT.
 * **audiences** - The list of JWT [audiences](https://tools.ietf.org/html/rfc7519#section-4.1.3){:target="_blank"}. that are allowed to access. A JWT containing any of these audiences will be accepted.

Then execute the following oc command in CodeReady Workspaces Terminal to create this object:

`oc create -f /projects/cloud-native-workshop-v2m3-labs/catalog/rules/ccn-auth-config.yaml -n userXX-catalog` (replace `userXX` with your username!)

Now you can't access the catalog service without authentication of RH-SSO. You confirm it using a curl command  (replacing `userXX` with your username) in CodeReady Workspaces Terminal:

`curl -i http://YOUR_CATALOG_GATEWAY_URL/services/products ; echo`

You should get and `HTTP 401 Unauthorized` and `Origin authentication failed.` messages.

The expected response is here because the user has not been identified with a valid JWT token in RH-SSO.
It normally takes `5 ~ 10 seconds` to initialize the authentication policy in Istio Mixer. After this things go quickly as policies are cached for some period of time.


![sso]({% image_path rhsso_call_catalog_noauth.png %})

In order to generate a correct token, run next `curl` request in CodeReady Workspaces Terminal. This command will
store the output Authorization token from RH-SSO in an environment variable called **TOKEN**.

> Replace `YOUR_SSO_HTTP_ROUTE_URL` with your own HTTP route url of SSO container that you created earlier.

> Also replace `authuserXX` with your authentication username, e.g. `authuser34`

~~~shell
export TOKEN=$( curl -X POST 'http://YOUR_SSO_HTTP_ROUTE_URL/auth/realms/istio/protocol/openid-connect/token' \
 -H "Content-Type: application/x-www-form-urlencoded" \
 -d "username=authuserXX" \
 -d 'password=openshift' \
 -d 'grant_type=password' \
 -d 'client_id=ccn-cli' | jq -r '.access_token')
~~~

Ensure you have a valid token:

`echo; echo $TOKEN; echo`

Once you have generated the token, re-run the curl command below with the token in CodeReady Workspaces Terminal:

`curl -H "Authorization: Bearer $TOKEN" http://YOUR_CATALOG_GATEWAY_URL/services/products ; echo`

You will see the following expected output:

~~~json
[{"itemId":"329299","name":"Red Fedora","desc":"Official Red Hat Fedora","price":34.99,"quantity":736},{"itemId":"329199","name":
"Forge Laptop Sticker","desc":"JBoss Community Forge Project Sticker","price":8.5,"quantity":512},{"itemId":"165613","name":"Solid
Performance Polo","desc":"Moisture-wicking, antimicrobial 100% polyester design wicks for life of garment. No-curl, rib-knit collar;
special collar band maintains crisp fold; three-button placket with dyed-to-match buttons; hemmed sleeves; even bottom with side vents;
Import. Embroidery. Red Pepper.","price":17.8,"quantity":256},{"itemId":"165614","name":"Ogio Caliber Polo","desc":"Moisture-wicking 100%
polyester. Rib-knit collar and cuffs; Ogio jacquard tape inside neck; bar-tacked three-button placket with Ogio dyed-to-match buttons;
...
~~~

![sso]({% image_path rhsso_call_catalog_auth.png %})

Congratulations! You've integrated RH-SSO with Istio to protect service mesh traffic to the catalog service, without having to change the application at all. Let's do it again with Spring Boot!

####7. Securing Spring Boot with Red Hat Single Sing-On

---

Unfortunately, the catalog service still doesn't work when you access via the web page because the application has no authentication configuration yet:

![sso]({% image_path rhsso_web_catalog_noauth.png %})

Let's integrate RH-SSO authentication to the presentation layer of the catalog service.
First, clean up all authentication configuration that we have tested in the previous steps. Run the following script to clean up:

`/projects/cloud-native-workshop-v2m3-labs/istio/scripts/cleanup.sh userXX` (replace `userXX` with your username!)

Next, open the **application-default.properties** in `/projects/cloud-native-workshop-v2m3-labs/catalog/src/main/resources/` and add the following settings at the bottom of the file:

Replace **YOUR_SSO_HTTP_ROUTE_URL/**

~~~yaml
#TODO: Set RH-SSO authentication
keycloak.auth-server-url=http://YOUR_SSO_HTTP_ROUTE_URL/auth
keycloak.realm=istio
keycloak.resource=ccn-cli
keycloak.public-client=true

keycloak.security-constraints[0].authRoles[0]=ccn_auth
keycloak.security-constraints[0].securityCollections[0].patterns[0]=/*
~~~

Let's update **pom.xml** in `/projects/cloud-native-workshop-v2m3-labs/catalog/` to add the needed keycloak dependency to our app:.

 * Add _spring-boot-starter-parent_ artifact Id before _properties_ element:

~~~xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.21.RELEASE</version>
        <relativePath/>
    </parent>
~~~

![sso]({% image_path rhsso_catalog_pom_parent.png %})

 * Replace **me.snowdrop** dependencyManagement and **spring-boot-starter** dependency with _keycloak_ dependency.

**From:**

~~~yaml
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>me.snowdrop</groupId>
                <artifactId>spring-boot-bom</artifactId>
                <version>${spring-boot.bom.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
~~~

**To:**

~~~yaml
    <dependencyManagement>
       <dependencies>
          <dependency>
              <groupId>org.keycloak.bom</groupId>
              <artifactId>keycloak-adapter-bom</artifactId>
              <version>3.1.0.Final</version>
              <type>pom</type>
              <scope>import</scope>
          </dependency>
      </dependencies>
    </dependencyManagement>
    <dependencies>
        <dependency>
          <groupId>org.keycloak</groupId>
          <artifactId>keycloak-spring-boot-starter</artifactId>
        </dependency>
~~~

![sso]({% image_path rhsso_catalog_pom_dependency.png %})

Let's re-deploy the catalog service to OpenShift by running the following maven command in CodeReady Workspaces Terminal:

`cd /projects/cloud-native-workshop-v2m3-labs/catalog`

`mvn clean package spring-boot:repackage -DskipTests`

`oc -n userXX-catalog start-build catalog-springboot --from-file=target/catalog-1.0.0-SNAPSHOT.jar --follow` (replace `userXX` with your username)

Wait for the catalog pod to restart:

`oc rollout status -w dc/catalog-springboot -n userXX-catalog`  (replace `userXX` with your username)

After the catalog pod is started, access the _catalog gateway_ via a new web brower then you will redirect to the login page of **RH-SSO**.

Input the following credential that we created it in RH-SSO administration page eariler.

 * Username or email: **authuserXX** (replace with your auth user, e.g. `authuser34`)
 * Password: **openshift**

![sso]({% image_path rhsso_catalog_redirect.png %})

Finally, you can access the catalog service as below:

![sso]({% image_path rhsso_web_catalog_auth.png %})

#### Summary

In this scenario you used Istio to implement many of the features needed in modern, distributed applications.

Istio provides an easy way to create a network of deployed services with load balancing, service-to-service authentication, monitoring, and more
without requiring any changes in service code. You add Istio support to services by deploying a special sidecar proxy throughout your environment
that intercepts all network communication between microservices, configured and managed using Istio’s control plane functionality.

Technologies like containers and container orchestration platforms like OpenShift solve the deployment of our distributed
applications quite well, but are still catching up to addressing the service communication necessary to fully take advantage
of distributed microservice applications. With Istio you can solve many of these issues outside of your business logic,
freeing you as a developer from concerns that belong in the infrastructure. **Congratulations!**