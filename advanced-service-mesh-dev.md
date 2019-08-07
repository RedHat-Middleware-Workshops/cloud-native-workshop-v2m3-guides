## Lab3 - Advanced Service Mesh Development

In this lab, you will develop advanced servie mesh features such as `Fault Injection`, `Traffic Shifting`, `Circuit Breaker`,
`Rate Limit` with `Coolstore microservices`(i.e Catalog, Inventory) that you developed and deployed to OpenShift cluster 
in `Module 1` or/and `Module 2`.

If you haven't deployment in Module 1 or Module2, you can deploy the cloud-native applications easily via executing the following shell script in CodeReady Workspace Terminal:

`chmod +x /projects/cloud-native-workshop-v2m3-labs/istio/scripts/deploy-*.sh`

Replace with your username before running this commands:

`/projects/cloud-native-workshop-v2m3-labs/istio/scripts/deploy-inventory.sh userXX`

`/projects/cloud-native-workshop-v2m3-labs/istio/scripts/deploy-catalog.sh userXX`

####1. Configuring Automatic Sidecar Injection in Coolstore Microservices

Lets' go to `Kiali console` once again to confirm if existing microservices(`Catalog`, `Inventory`) are running with a `side car`.
Click on `Applictions` on the left menu check `userXX-catalog`, `userXX-inventory` in namespaces. You will see `Missing Sidecar` in 4 applications. 

![istio]({% image_path kiali_missing_sidecar.png %})

Upstream Istio community installations rely on the existence of a `proxy sidecar` within the application’s pod to provide service mesh capabilities to the application. You can include the proxy sidecar by using a manual process before deployment. However, automatic injection ensures that your application contains the appropriate configuration for your service mesh at the time of its deployment.

`Automatic injection of the sidecar` is supported by using the `sidecar.istio.io/inject` annotation within your application yaml file. Set the annotation’s value to true for injection to occur.

> Upstream Istio community installations require a specific label on the namespace after which all pods in that namespace are injected with the sidecar.  The OpenShift Service Mesh approach requires you to opt in to injection using an annotation with no need to label namspaces. This method requires fewer privileges and does not conflict with other OpenShift capabilities such as builder pods.

Go to `Workloads > Deployment Configs` on the left menu, select `userXX-inventory` project and click on `inventory-database`. 

![istio]({% image_path inventory_db_dc.png %})

Add the following annotation in `spec.template.metadata.annotations` path and click on `Save`.

`sidecar.istio.io/inject: "true"`

![istio]({% image_path inventory_db_inject_sidecar.png %})

You will see `istio-proxy` container and `inventory-database` container in Pod Details page when you navigate `Workloads > Pods` > `inventory-database-xxxxx`.

![istio]({% image_path inventory_db_sidecar.png %})

Now you will inject a sidecar container to application container(Inventory) as well, navigate `Workloads > Deployment Configs` on the left menu. 
Select `userXX-inventory` project and click on `inventory-quarkus`.

![istio]({% image_path inventory_dc.png %})

`sidecar.istio.io/inject: "true"`

![istio]({% image_path inventory_inject_sidecar.png %})

You will see `istio-proxy` container and `inventory-quarkus` container in Pod Details page when you navigate `Workloads > Pods` > `inventory-quarkus-xxxxx`:

![istio]({% image_path inventory_sidecar.png %})

Next, go to `Workloads > Deployment Configs` on the left menu, select `userXX-catalog` project and click on `catalog-database`. 

![istio]({% image_path catalog_db_dc.png %})

Then click on `YAML` tab and add the following annotation in `spec.template.metadata.annotations` path and click on `Save`.

`sidecar.istio.io/inject: "true"`

![istio]({% image_path catalog_db_inject_sidecar.png %})

You will see `istio-proxy` container and `catalog-database` container in Pod Details page when you navigate `Workloads > Pods` > `catalog-database-xxxxx`.

![istio]({% image_path catalog_db_sidecar.png %})

Now you will inject a sidecar container to application container(Catalog) as well, 
go to `Workloads > Deployment Configs` on the left menu, select `userXX-catalog` project and click on `catalog-springboot`. 

![istio]({% image_path catalog_dc.png %})

`sidecar.istio.io/inject: "true"`

![istio]({% image_path catalog_inject_sidecar.png %})

You will see `istio-proxy` container and `catalog-springboot` container in Pod Details page when you navigate `Workloads > Pods` > `catalog-springboot-xxxxx`:

![istio]({% image_path catalog_sidecar.png %})

Let's make sure if inventory and catalog services are working correctly via accessing `Catalog Route URL`:

`i.e. http://catalog-springboot-user0-catalog.apps.cluster-seoul-a30e.seoul-a30e.openshiftworkshop.com/`

You will see the following web page including `Inventory Quantity` if the catalog service can access the inventory service via `Istio proxy sidecar`:

![istio]({% image_path catalog_route_sidecar.png %})

> Do not close the above `Catalog UI browser` to create traffics between services because this page continues to invoke catalog service and inventory service.

Now, reload `Applications` in `Kiali console` to check if the `Missing sidecar` doesn't show any longer:

![istio]({% image_path kiali_injecting_sidecar.png %})

Also, go to the Service Graph page and check `userXX-inventory`, `userXX-catalog` in Namespace, check `Traffic Animation` in `Display` for understanding 
the traffic flow from catalog service to inventory service:

![istio]({% image_path kiali_graph_sidecar.png %})

####2. Fault Injection

---

This step will walk you through how to use `fault injection` to test the end-to-end failure recovery capability of the application as a whole. An incorrect configuration of the failure recovery policies could result in unavailability of critical services. Examples of incorrect configurations include incompatible or restrictive timeouts across service calls.

`Istio` provides a set of failure recovery features that can be taken advantage of by the services
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

Two types of faults can be injected: 

 * `Delays` are timing failures. They mimic increased network latency or an overloaded upstream service.
 * `Aborts` are crash failures. They mimic failures in upstream services. Aborts usually manifest in the form of HTTP error codes or TCP connection failures.


##### Inject a fault

To test our application microservices for resiliency, we will inject a failure(`500 status`) in `50%` of requests to `inventory` microservices.

Remove the route that we exposed the inventory service to manage network traffic by `Istio Ingressgateway`. Use the following command for `your own route name` at CodeReady Workspace `Terminal`:

> Copy the route URL(i.e. inventory-quarkus-user1-inventory.apps.seoul-bfcf.openshiftworkshop.com) and you will reuse the URL to create a gateway in Istio.

`oc delete route/inventory-quarkus -n userXX-inventory`

Add the following label in the Inventory service to use a `virtural service` via OpenShift Web Consle 
when you navigate `Networking > Services` in the left menu. Select `userXX-inventory` project and click on `inventory-quarkus`.

![fault-injection]({% image_path inventory_svc_.png %})

Click on `YAML` tab and add the following variables.

`service: inventory-quarkus`

![fault-injection]({% image_path inventory_svc_add_label.png %})

Click on `Save`.

Open a `inventory-default.yaml` file in `/projects/cloud-native-workshop-v2m3-labs/inventory/rules/` to make a gateway and virtual service:

> You need to replace all `YOUR_INVENTORY_GATEWAY_URL` with the previous route URL that you copied earlier.

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

Run the following command via CodeReady Workspace `Terminal`:

`oc create -f /projects/cloud-native-workshop-v2m3-labs/inventory/rules/inventory-default.yaml -n userXX-inventory`

Now, you can test if the inventory service works correctly via accessing the `gateway URL`:

`i.e. http://inventory-quarkus-user0-inventory.apps.cluster-seoul-a30e.seoul-a30e.openshiftworkshop.com`

![fault-injection]({% image_path inventory-ui-gateway.png %})

Let's inject a failure(`500 status`) in `50%` of requests to `inventory` microservices. Edit `inventory-default.yaml` as below.

Open `inventory-vs-fault.yaml` file in `/projects/cloud-native-workshop-v2m3-labs/inventory/rules/` and copy the following codes.

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

Before creating a new `inventory-fault VirtualService`, we need to delete the existing `inventory-default VirtualService`.
Run the following command via CodeReady Workspace `Terminal`:

`oc delete virtualservice/inventory-default -n userXX-inventory`

Then create a new virtualservice and gateway.

`oc create -f /projects/cloud-native-workshop-v2m3-labs/inventory/rules/inventory-vs-fault.yaml -n userXX-inventory`

Let's find out if the fault injection works corectly via accessing the Inventory gateway once again. You will see that the `Status` of CoolStore Inventory continues to change between `DEAD` and `OK`:

![fault-injection]({% image_path inventory-dead-ok.png %})

To make sure if the `50%` traffic is failed with `500 Error` in `Kiali Graph`. You will see `red` traffic from `istio-ingressgateway` as well as around 50% of requests are displayed as `5xx` on the right side, `HTTP Traffic`. The reason why the error rate is not exact 50% is that the request keeps coming from catalog and ingress gateway at the same time.

![fault-injection]({% image_path inventlry-vs-error-kiali.png %})

Let's make another injection in terms of you will introduce a `5 second delay` in `100% of requests` to Inventory service. Go to `Virtual Service` in `Other Resources` in OpenShift Web Console and Click on `Edit YAML` in inventory-default:

Open `inventory-vs-fault-delay.yaml` file in `/projects/cloud-native-workshop-v2m3-labs/inventory/rules/` and copy the following codes.

> You need to replace all `YOUR_INVENTORY_GATEWAY_URL` with the previous route URL that you copied earlier.

Before creating a new `inventory-fault-delay VirtualService`, we need to delete the existing `inventory-fault VirtualService`.
Run the following command via CodeReady Workspace `Terminal`:

`oc delete virtualservice/inventory-fault -n userXX-inventory`

Then create a new virtualservice and gateway.

`oc create -f /projects/cloud-native-workshop-v2m3-labs/inventory/rules/inventory-vs-fault-delay.yaml -n userXX-inventory`

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

When we go to `Kiali Graph`, you will see that the `green` traffic from `istio-ingressgateway` is delayed than requests from catalog service. Note that you need to check `Traffic Animation` in Display select box.

![fault-injection]({% image_path inventlry-vs-delay-kiali.png %})

If the Inventory’s front page was set to correctly handle delays, we expect it to load within
approximately 5 seconds. To see the web page response times, open the Developer Tools menu in
IE, Chrome or Firefox (typically, key combination `Ctrl`+`Shift`+`I` or `Alt`+`Cmd`+`I`), tab Network,
and reload the bookinfo web page.

You will see and feel that the webpage loads in about 5 seconds:

![Delay]({% image_path inventory-webui-delay.png %})

Before we will move to the next step, clean up the fault injection with the default virtual service as here.

`oc delete virtualservice/inventory-fault-delay -n userXX-inventory`

`oc delete gateway/inventory-gateway -n userXX-inventory`

`oc create -f /projects/cloud-native-workshop-v2m3-labs/inventory/rules/inventory-default.yaml -n userXX-inventory`


####3. Enable Circuit Breaker

---

In this step, you will configure an Istio Circuit Breaker to protect the calls `Inventory` service.
If the `Inventory` service gets overloaded due to call volume, Istio (in conjunction with Kubernetes) will limit
future calls to the service instances to allow them to recover.

Circuit breaking is a critical component of distributed systems.
It’s nearly always better to fail quickly and apply back pressure downstream
as soon as possible. Istio enforces circuit breaking limits at the network
level as opposed to having to configure and code each application independently.

Istio supports various types of circuit breaking:

* `Cluster maximum connections`: The maximum number of connections that Istio will establish to all hosts in a cluster.
* `Cluster maximum pending requests`: The maximum number of requests that will be queued while waiting for a
ready connection pool connection.
* `Cluster maximum requests`: The maximum number of requests that can be outstanding to all hosts in a
cluster at any given time. In practice this is applicable to HTTP/2 clusters since HTTP/1.1 clusters are
governed by the maximum connections circuit breaker.
* `Cluster maximum active retries`: The maximum number of retries that can be outstanding to all hosts
in a cluster at any given time. In general Istio recommends aggressively circuit breaking retries so that
retries for sporadic failures are allowed but the overall retry volume cannot explode and cause large
scale cascading failure.

> Note that `HTTP2` uses a single connection and never queues (always multiplexes), so max connections and
max pending requests are not applicable.

Each circuit breaking limit is configurable and tracked on a per upstream cluster and per priority basis.
This allows different components of the distributed system to be tuned independently and have different limits.
See the [Envoy’s circuit breaker](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking) for more details.

Let's add a circuit breaker to the calls to the `Inventory` service. Instead of using a _VirtualService_ object,
circuit breakers in isto are defined as _DestinationRule_ objects. DestinationRule defines policies that apply to traffic intended for a service after routing has occurred. These rules specify configuration for load balancing, connection pool size from the sidecar, and outlier detection settings to detect and evict unhealthy hosts from the load balancing pool.

Open a `inventory-cb.yaml` file in `/projects/cloud-native-workshop-v2m3-labs/inventory/rules/` to apply 
circuit breaking settings when calling the `Inventory` service:

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

> If you installed/configured Istio with `mutual TLS` authentication enabled, you must add a TLS traffic policy `mode: ISTIO_MUTUAL` to the DestinationRule before applying it. 

![circuit-breaker]({% image_path inventory-circuit-breaker.png %})

Run the following command via CodeReady Workspace `Terminal`:

`oc create -f /projects/cloud-native-workshop-v2m3-labs/inventory/rules/inventory-cb.yaml -n userXX-inventory`

We set the `Inventory` service's maximum connections to 1 and maximum pending requests to 1. Thus, if we send more
than 2 requests within a short period of time to the reviews service, 1 will go through, 1 will be pending,
and any additional requests will be denied until the pending request is processed. Furthermore, it will detect any hosts that
return a server error (5XX) and eject the pod out of the load balancing pool for 15 minutes. You can visit
here to check the
[Istio spec](https://istio.io/docs/reference/config/traffic-rules/destination-policies.html#istio.proxy.v1.config.CircuitBreaker.SimpleCircuitBreakerPolicy)
for more details on what each configuration parameter does.

####4. Overload the service

---

Let's use some simple `curl` commands to send multiple concurrent requests to our application, and witness the
circuit breaker kicking in opening the circuit.

Execute this to simulate a number of users attampting to access the gateway URL simultaneously:

Your Inventory gateway URL seems like `http://inventory-quarkus-user0-inventory.apps.cluster-seoul-a30e.seoul-a30e.openshiftworkshop.com`

~~~shell
    for i in {1..50} ; do
        curl 'http://YOUR_IVENTORY_GATEWAY_URL/services/inventory' >& /dev/null &
    done
~~~

Due to the very conservative circuit breaker, many of these calls will fail with HTTP 503 (Server Unavailable). To see this,
open the `Istio Service Mesh Dashboard` in Grafana console and select `inventory-quarkus.userxx-inventory.svc.cluster.local` service:

> `NOTE`: It make take 10-20 seconds before the evidence of the circuit breaker is visible
within the Grafana dashboard, due to the not-quite-realtime nature of Prometheus metrics and Grafana
refresh periods and general network latency.

![circuit-breaker]({% image_path inventory-circuit-breaker-grafana.png %})

That's the circuit breaker in action, limiting the number of requests to the service. In practice your limits would be much higher.

####5. Stop overloading

---

Before moving on, stop the traffic generator by executing the following commands in CodeReady Workspace `Terminal`:

`for i in {1..50} ; do kill %${i} ; done`

![circuit-breaker]({% image_path inventory-circuit-breaker-stop.png %})

Delete the circuit breaker of the Inventory service via the following commands. You should replace `userxx` with your namespace:

`oc delete destinationrule/inventory-cb -n userxx-inventory`

####6. Enable Authentication using Single Sign-on

---

In this step, you will learn how to enable authenticating `catalog` microservices with Istio, [JSON Web Token(JWT)](https://en.wikipedia.org/wiki/JSON_Web_Token), and 
[Red Hat Single Sign-On](https://access.redhat.com/products/red-hat-single-sign-on) in [Red Hat Application Runtimes](https://www.redhat.com/en/products/application-runtimes).

First, let's remove the route that we exposed the catalog service to manage network traffic by `Istio Ingressgateway`. 
Use the following command for `your own route name` at CodeReady Workspace `Terminal`:

> Copy the route URL(i.e. catalog-user1-catalog.apps.seoul-0993.openshiftworkshop.com) and you will reuse the URL to create a gateway in Istio.

`oc delete route/catalog-springboot -n userXX-catalog`

Add the following label in the catalog service to use a `virtural service` via OpenShift Web Consle when you navigate `Networking > Services` in the left menu.
Select `userXX-catalog` project and click on `catalog-springboot` service.

![sso]({% image_path catalog_svc_vs.png %})

Click on `YAML` tab and add the following label.

`service: catalog`

![sso]({% image_path catalog_svc_add_label.png %})

Click on `Save`.

Open a `catalog-default.yaml` file in `/projects/cloud-native-workshop-v2m3-labs/catalog/rules/` to make a gateway and virtual service:

> Replace all `YOUR_CATALOG_GATEWAY_URL` with the previous route URL that you copied earlier.

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

Run the following command via CodeReady Workspace `Terminal`:

`oc create -f /projects/cloud-native-workshop-v2m3-labs/catalog/rules/catalog-default.yaml -n userXX-catalog`

Now, you can test if the inventory service works correctly via accessing the `YOUR_CATALOG_GATEWAY_URL` without `authentication`:

`i.e. http://catalog-user1-catalog.apps.seoul-6eb1.openshiftworkshop.com`

![sso]({% image_path catalog-ui-gateway.png %})

Let's deploy `Red Hat Single Sign-On (RH-SSO)` to provide security token that enables service authentication in Istio.

`Red Hat Single Sign-On (RH-SSO)` is based on the `Keycloak` project and enables you to secure your web applications by providing 
Web single sign-on (SSO) capabilities based on popular standards such as `SAML 2.0, OpenID Connect and OAuth 2.0`. The RH-SSO server 
can act as a SAML or OpenID Connect-based Identity Provider, mediating with your enterprise user directory or 3rd-party SSO provider 
for identity information and your applications via standards-based tokens. The major features are here:

 * `Authentication Server` - Acts as a standalone SAML or OpenID Connect-based Identity Provider.
 * `User Federation` - Certified with LDAP servers and Microsoft Active Directory as sources for user information.
 * `Identity Brokering` - Integrates with 3rd-party Identity Providers including leading social networks as identity source.
 * `REST APIs and Administration GUI` - Specify user federation, role mapping, and client applications with easy-to-use Administration GUI and REST APIs.

We will deploy RH-SSO in Catalog project via running the following commands in CodeReady Workspace `Terminal`:

You need to replace your username with `authuserXX`.

~~~shell
oc -n userXX-catalog new-app ccn-sso72 \
   -p SSO_ADMIN_USERNAME=admin \
   -p SSO_ADMIN_PASSWORD=admin \
   -p SSO_REALM=istio \
   -p SSO_SERVICE_USERNAME=authuserXX \
   -p SSO_SERVICE_PASSWORD=openshift
~~~

> If you change `SSO_ADMIN_USERNAME`, `SSO_ADMIN_PASSWORD` then you need to login RH-SSO web console with them.

Once you complete to deploy `RH-SSO` in `Networking > Routes` at OpenShift web console then you will see `HTTPS/HTTP` route URL as below:

![sso]({% image_path rhsso_deployment.png %})

Click on `HTTPS` URL(i.e. _secure-sso-user0-catalog.apps.cluster-seoul-a30e.seoul-a30e.openshiftworkshop.com_) to access RH-SSO web console as below:

![sso]({% image_path rhsso_landing_page.png %})

Click on `Administration Console` to configure *Istio* Ream then input usename and password that you deployed.

 * Username or email: `admin`
 * Password: `admin`

 > If you change the credential when you deployed the RH-SSO container, you need to use them for this.

![sso]({% image_path rhsso_admin_login.png %})

You will see general information of `Istio Realm Setting` and click on `Login` tab to swich off `Require SSL` with `none` then click on `Save`.

![sso]({% image_path rhsso_istio_realm.png %})

> Red Hat Single Sign-On generates a self-signed certificate the first time it runs. Please note that self-signed certificates doesn't work to authenticate by Istio 
so we will change not to use SSL for testing Istio authentication.

Next, create a new `client` that is trusted browser apps and web services in a `Istio` realm. The client can request a login. You can also define client specific roles.
Go to `Clients` in the left menu then click on `Create`.

![sso]({% image_path rhsso_clients.png %})

Input `ccn-cli` in `Client ID` field and click on `Save`.

![sso]({% image_path rhsso_clients_create.png %})

On the next screen, you will see details of the `Settings` tab, the only thing you need to do is to input `Valid Redirect URIs` after successful login or logout.

> Replace `YOUR_CATALOG_GATEWAY_URL` with your own ingress gatewat URL of the catalog service and please note to add `/*` at the end of URL.

 * Valid Redirect URIs: `http://YOUR_CATALOG_GATEWAY_URL/*`

![sso]({% image_path rhsso_clients_settings.png %})

Now, you will define a role that will be assigned to your credential, let’s create a simple role called `ccn_auth`.
Go to `Roles` in the left menu then click on `Add Role`.

![sso]({% image_path rhsso_roles.png %})

Input `ccn_auth` in `Role Name` field and click on `Save`.

![sso]({% image_path rhsso_roles_create.png %})

And update the password of your credentials(i.e. authuserXX) that is created automatically when you deployed RH-SSO earlier. 
Because the password doesn't set as defual when you create a credential in RH-SSO. Go to `Users` menu on the left side menu then click on `View all users`.

![sso]({% image_path rhsso_users.png %})

If you click on ID then you will find more information such as Detials, Attributes, Credentials, Role Mappings, Groups, 
Contents, and Sessions. You don't need to update any details in this step. 

![sso]({% image_path rhsso_istio_users_details.png %})

Go to `Credentials` tab and input the following variables:

 * New Password: `r3dh4t1!`
 * Password Confirmation: `r3dh4t1!`
 * Temporary: `OFF`

Make sure to turn off the “Temporary” flag unless you want the authuserXX to have to change his password the first time he authenticates.

Click on `Reset Password`.

![sso]({% image_path rhsso_users_credentials.png %})

Then click on `Change password` in the popup window.

![sso]({% image_path rhsso_users_change_pwd.png %})

Now proceed to the `Role Mappings` tab and assign the role `ccn_auth` via clicking on `Add selected >`.

![sso]({% image_path rhsso_rolemapping.png %})

You will confirm the ccn_auth role in `Assigned Roles` box.

![sso]({% image_path rhsso_rolemapping_assigned.png %})

Well done to enable RH-SSO server! Let's create an user-facing authentication policy using JSON Web Token(JWT) token.
JWT token format for authentication as defined by [RFC 7519](https://tools.ietf.org/html/rfc7519). You can find more details 
how [OAuth 2.0](https://tools.ietf.org/html/rfc6749) and [OIDC 1.0](https://openid.net/connect/) works in the whole authentication flow.

Open a `ccn-auth-config.yml` file in `/projects/cloud-native-workshop-v2m3-labs/catalog/rules/` to create an authentication policy:

> Replace all `YOUR_SSO_HTTP_ROUTE_URL` with your own HTTP route url of SSO container that you created earlier and also replace `USERXX` with your username. 

You can also get the route url via executing the following commands in CodeReady Workspace `Terminal`:

`oc get route -n userXX-catalog | grep -v secure | awk 'NR>1{print $2}' | grep sso`

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

You can also define the following fields to create a Policy in Istio.

 * `issuer` - Identifies the issuer that issued the JWT. See [issuer](https://tools.ietf.org/html/rfc7519#section-4.1.1) usually a URL or an email address.
 * `jwksUri` - URL of the provider’s public key set to validate signature of the JWT.
 * `audiences` - The list of JWT [audiences](https://tools.ietf.org/html/rfc7519#section-4.1.3). that are allowed to access. A JWT containing any of these audiences will be accepted.

Then execute the following oc command in CodeReady Workspace `Terminal`:

`oc create -f /projects/cloud-native-workshop-v2m3-labs/catalog/rules/ccn-auth-config.yaml`

Now you can't access the catalog service without authentication of RH-SSO. You confirm it using CURL command with replacing USERXX in CodeReady Workspace `Terminal`:

`curl http://YOUR_CATALOG_GATEWAY_URL/services/products ; echo`

The expected response is here because the user has not been identified with a valid JWT token in RH-SSO. 
It normally takes `5 ~ 10 seconds` to apply the authentication policy in Istio Mixer.

> Origin authentication failed.

![sso]({% image_path rhsso_call_catalog_noauth.png %})

In order to generate a correct token, just run next curl request in CodeReady Workspace `Terminal`. This command will 
store the output Authorization token from RH-SSO in an environment variable called `$TOKEN`. 

> Replace `YOUR_SSO_HTTP_ROUTE_URL` with your own HTTP route url of SSO container that you created earlier. 

~~~shell
export TOKEN=$( curl -X POST 'http://YOUR_SSO_HTTP_ROUTE_URL/auth/realms/istio/protocol/openid-connect/token' \
 -H "Content-Type: application/x-www-form-urlencoded" \
 -d "username=authuser1" \
 -d 'password=openshift' \
 -d 'grant_type=password' \
 -d 'client_id=ccn-cli' | jq -r '.access_token')
~~~

Once you have generated the token, re-run the curl command below with the token in CodeReady Workspace `Terminal`:

`curl -H "Authorization: Bearer $TOKEN" http://YOUR_CATALOG_GATEWAY_URL/services/products ; echo`

You will see the following expected output:

~~~shell
[{"itemId":"329299","name":"Red Fedora","desc":"Official Red Hat Fedora","price":34.99,"quantity":736},{"itemId":"329199","name":
"Forge Laptop Sticker","desc":"JBoss Community Forge Project Sticker","price":8.5,"quantity":512},{"itemId":"165613","name":"Solid 
Performance Polo","desc":"Moisture-wicking, antimicrobial 100% polyester design wicks for life of garment. No-curl, rib-knit collar; 
special collar band maintains crisp fold; three-button placket with dyed-to-match buttons; hemmed sleeves; even bottom with side vents;
Import. Embroidery. Red Pepper.","price":17.8,"quantity":256},{"itemId":"165614","name":"Ogio Caliber Polo","desc":"Moisture-wicking 100% 
polyester. Rib-knit collar and cuffs; Ogio jacquard tape inside neck; bar-tacked three-button placket with Ogio dyed-to-match buttons; 
...
~~~

![sso]({% image_path rhsso_call_catalog_auth.png %})


####7. Securing Spring Boot with Red Hat Single Sing-On

---

However, the catalog service doesn't still work when you access to the web page as below because the applicaion has no authentication codes, configuration yet:

![sso]({% image_path rhsso_web_catalog_noauth.png %})

Let's integrate RH-SSO authentication to the presentation layer of the catalog service. 
First, clean up all authentication configuration that we have tested in the previous steps. Run the following script to clean up in CodeReady Workspace `Terminal`:

`/projects/cloud-native-workshop-v2m3-labs/istio/scripts/cleanup.sh userXX`

Next, open `application-openshift.properties` in `/projects/cloud-native-workshop-v2m3-labs/catalog/src/main/resources/` and add the following settings:

~~~yaml
#TODO: Set RH-SSO authentication
keycloak.auth-server-url=http://YOUR_SSO_HTTP_ROUTE_URL/auth
keycloak.realm=istio
keycloak.resource=ccn-cli
keycloak.public-client=true

keycloak.security-constraints[0].authRoles[0]=ccn_auth
keycloak.security-constraints[0].securityCollections[0].patterns[0]=/*
~~~

Let's update `pom.xml' in `/projects/cloud-native-workshop-v2m3-labs/catalog/` to process keycloak configuration by Spring Boot.

 * Add `spring-boot-starter-parent` artifact Id before `properties` element:

~~~yaml
<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.21.RELEASE</version>
		<relativePath/> 
</parent>  
~~~

![sso]({% image_path rhsso_catalog_pom_parent.png %})

 * Replace `nme.snowdrop` dependencyManagement and `spring-boot-starter` dependency with `keycloak` dependency.

`From:`

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

`To:`

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

Let's re-deploy the catalog service to OpenShift via running the following maven command in CodeReady Workspace `Terminal`:

`cd /projects/cloud-native-workshop-v2m3-labs/catalog`

`mvn package fabric8:deploy -Popenshift -DskipTests`

Once you complete to build successfully, you have delete to `the route, healthcheck` that you already deleted because Fabric8 deployment recreates them automatically.
Execute the following `oc commands` to remove them:

`oc delete route/catalog`

`oc set probe dc/catalog --remove --readiness --liveness`

After the catalog pod get started completely, access the catalog gateway via a new web brower then you will redirect to the login page of `RH-SSO`.

Input the following credential that we created it in RH-SSO administration page eariler.

 * Username or email: `authuserXX`
 * Password: `openshift`

![sso]({% image_path rhsso_catalog_redirect.png %})

Finally, you can access to the catalog service as below:

![sso]({% image_path rhsso_web_catalog_auth.png %})

####8. Improving Web Login Features with Red Hat Single Sing-On

---

RH-SSO allows system admin to configure various login features in terms of User registration, Email as username, Forgot password,Remember Me, Login with email, and more.
Let's update the Login page with adding `User registration` and `Forgot password`.

Change from `Istio` to `Master` realm in the left top menu, click on `Login` tab. Next, toggle on `User registration` and `Forgot password` as below:

![sso]({% image_path rhsso_master_realm_change.png %})

Don't forget to click on `Save`. Let's confirm the changed login page via clicking on `Sign Out` in the right top menu as below:

![sso]({% image_path rhsso_master_signout.png %})

The page will be redirected automatically to the login page. Now you will see `Forget Password?` link under the password field. 
You can find `New User? Register` link on the right side.

![sso]({% image_path rhsso_master_relogin.png %})

If you click on `Forget Password?` link, you will the below the page where you can change the password of your credential.

![sso]({% image_path rhsso_master_forgot_pwd.png %})

When you click on `Register` link, you will the below the page where you can create a new credential.

![sso]({% image_path rhsso_master_reg_user.png %})

#### Summary

In this scenario you used Istio to implement many of the
Istio provides an easy way to create a network of deployed services with load balancing, service-to-service authentication, monitoring, and more 
without requiring any changes in service code. You add Istio support to services by deploying a special sidecar proxy throughout your environment 
that intercepts all network communication between microservices, configured and managed using Istio’s control plane functionality.

Technologies like containers and container orchestration platforms like OpenShift solve the deployment of our distributed
applications quite well, but are still catching up to addressing the service communication necessary to fully take advantage
of distributed microservice applications. With Istio you can solve many of these issues outside of your business logic,
freeing you as a developer from concerns that belong in the infrastructure. `Congratulations!`