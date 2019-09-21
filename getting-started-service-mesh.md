## Service Mesh and Identity

In this module, we will learn how to prevent cascading failures in a distributed environment, how to detect misbehaving services, and how to avoid having to implement resiliency and monitoring in your business logic. As we transition our applications towards a distributed architecture with microservices deployed across a distributed
network, Many new challenges await us.

Technologies like containers and container orchestration platforms like OpenShift solve the deployment of our distributed
applications quite well, but are still catching up to addressing the service communication necessary to fully take advantage
of distributed applications, such as dealing with:

* Unpredictable failure modes
* Verifying end-to-end application correctness
* Unexpected system degradation
* Continuous topology changes
* The use of elastic/ephemeral/transient resources

Today, developers are responsible for taking into account these challenges, and do things like:

* Circuit breaking and Bulkheading (e.g. with Netfix Hystrix)
* Timeouts/retries
* Service discovery (e.g. with Eureka)
* Client-side load balancing (e.g. with Netfix Ribbon)

Another challenge is each runtime and language addresses these with different libraries and frameworks, and in
some cases there may be no implementation of a particular library for your chosen language or runtime.

In this scenario we'll explore how to use a new project called _Istio_ to solve many of these challenges and result in
a much more robust, reliable, and resilient application in the face of the new world of dynamic distributed applications.

#### What is Istio?

---

![Logo]({% image_path istio-logo.png %}){:width="600px"}

Istio is an open, platform-independent service mesh designed to manage communications between microservices and
applications in a transparent way.It provides behavioral insights and operational control over the service mesh
as a whole. It provides a number of key capabilities uniformly across a network of services:

* `Traffic Management` - Control the flow of traffic and API calls between services, make calls more reliable, and make the network more robust in the face of adverse conditions.

* `Observability` - Gain understanding of the dependencies between services and the nature and flow of traffic between them, providing the ability to quickly identify issues.

* `Policy Enforcement` - Apply organizational policy to the interaction between services, ensure access policies are enforced and resources are fairly distributed among consumers. Policy changes are made by configuring the mesh, not by changing application code.

* `Service Identity and Security` - Provide services in the mesh with a verifiable identity and provide the ability to protect service traffic as it flows over networks of varying degrees of trustability.

These capabilities greatly decrease the coupling between application code, the underlying platform, and policy. This decreased coupling not only makes services easier to implement, but also makes it simpler for operators to move application deployments between environments or to new policy schemes. Applications become inherently more portable as a result.

Sounds fun, right? Let's get started!

#### Examine Istio

---

For this module we've already installed Istio into our OpenShift platform.

You should note that you must be logged in as `cluster-admin` role if you want to install Istio on your own OpenShift cluster. This is required as this
user will need to run things in a privileged way, or even with containers as root.

For example, you can run the following to login as admin in OpenShift cluster:

`oc login <YOUR_OPENSHIFT_MASTER_CONSOLE_URL> -u admin -p admin --insecure-skip-tls-verify=true`

What's happend during Istio installation eariler as below:

* Creates the project `istio-system` as the location to deploy all the components
* Adds necessary permissions
* Deploys Istio components
* Deploys additional add-ons, namely `Kiali, Prometheus, Grafana, Service Graph and Jaeger Tracing`
* Exposes routes for those add-ons and for Istio's Ingress component

We'll use the above components througout this scenario, so don't worry if you don't know what they do!

You can also read a bit more about the [Istio](https://istio.io/docs) architecture below:

#### Istio Details

---

An Istio service mesh is logically split into a `data plane` and a `control plane`.

The `data plane` is composed of a set of intelligent proxies (_Envoy_ proxies) deployed as _sidecars_ to your application's pods in OpenShift that mediate and control all network communication between microservices.

The `control plane` is responsible for managing and configuring proxies to route traffic, as well as enforcing policies at runtime.

The following diagram shows the different components that make up each plane:

![Istio Arch]({% image_path arch.png %}){:width="800px"}

##### Istio Components

`Envoy`
Istio uses an extended version of the [Envoy](https://envoyproxy.github.io/envoy/) proxy. Envoy is a high-performance proxy developed in C++ to mediate all inbound and outbound traffic for all services in the service mesh. Istio leverages Envoy’s many built-in features, for example:

 * Dynamic service discovery
 * Load balancing
 * TLS termination
 * HTTP/2 and gRPC proxies
 * Circuit breakers
 * Health checks
 * Staged rollouts with %-based traffic split
 * Fault injection
 * Rich metrics

Envoy is the `data plane` component that deployed as a `sidecar` to the relevant service in the same Kubernetes pod. This deployment allows Istio to extract a wealth of signals about traffic behavior as attributes. Istio can, in turn, use these attributes in `Mixer` to enforce policy decisions, and send them to monitoring systems to provide information about the behavior of the entire mesh.

`Mixer`
Mixer is the `control plane` component responsible for enforcing access control and usage policies across the service mesh, and collects telemetry data from the Envoy proxy and other services. The proxy extracts request level attributes, and sends them to Mixer for evaluation.

Mixer includes a flexible plugin model. This model enables Istio to interface with a variety of host environments and infrastructure backends. Thus, Istio abstracts the Envoy proxy and Istio-managed services from these details.

`Pilot`
Pilot is the `control plane` component responsible for configuring the proxies at runtime. Pilot provides service discovery for the Envoy sidecars, traffic management capabilities for intelligent routing (for example, A/B tests or canary deployments), and resiliency (timeouts, retries, and circuit breakers).

Pilot converts high level routing rules that control traffic behavior into Envoy-specific configurations, and propagates them to the sidecars at runtime. Pilot abstracts platform-specific service discovery mechanisms and synthesizes them into a standard format that any sidecar conforming with the [Envoy data plane APIs](https://github.com/envoyproxy/data-plane-api) can consume. This loose coupling allows Istio to run on multiple environments such as Kubernetes, Consul, or Nomad, while maintaining the same operator interface for traffic management.

`Citadel`
Citadel is the `control plane` component responsible for certificate issuance and rotation. Citadel provides strong service-to-service and end-user authentication with built-in identity and credential management. You can use Citadel to upgrade unencrypted traffic in the service mesh. Using Citadel, operators can enforce policies based on service identity rather than on network controls.

`Galley`
Galley is Istio’s configuration validation, ingestion, processing and distribution component. It is responsible for insulating the rest of the Istio components from the details of obtaining user configuration from the underlying platform (e.g. Kubernetes).

`Add-ons`
Several components are used to provide additional visualizations, metrics, and tracing functions:

* [Kiali](https://www.kiali.io/) - Service mesh observability and configuration
* [Prometheus](https://prometheus.io/) - Systems monitoring and alerting toolkit
* [Grafana](https://grafana.com/) - Allows you to query, visualize, alert on and understand your metrics
* [Jaeger Tracing](http://jaeger.readthedocs.io/) - Distributed tracing to gather timing data needed to troubleshoot latency problems in microservice architectures
* [Servicegraph](https://istio.io/docs/tasks/telemetry/servicegraph.html#about-the-servicegraph-add-on) - generates and visualizes a graph of services within a mesh

We will use these in future steps in this scenario!

Check out the [Istio docs](https://istio.io/docs) for more details.

Is your Istio deployment complete? If so, then you're ready to move on!

#### Getting Ready for the labs

---

##### Access Your Development Environment

You will be using Red Hat CodeReady Workspaces, an online IDE based on [Eclipe Che](https://www.eclipse.org/che/){:target="_blank"}. **Changes to files are auto-saved every few seconds**, so you don't need to explicitly save changes.

To get started, [access the Che instance]({{ ECLIPSE_CHE_URL }}) and log in using the username and password you've been assigned (e.g. `{{ CHE_USER_NAME }}/{{ CHE_USER_PASSWORD }}`):

![cdw]({% image_path che-login.png %})

Once you log in, you'll be placed on your personal dashboard. We've pre-created workspaces for you to use. Click on the name of the pre-created workspace on the left, as shown below (the name will be different depending on your assigned number). You can also click on the name of the workspace in the center, and then click on the green button that says "OPEN" on the top right hand side of the screen:

![cdw]({% image_path che-precreated.png %})

After a minute or two, you'll be placed in the workspace:

![cdw]({% image_path che-workspace.png %})

To gain extra screen space, click on the yellow arrow to hide the left menu (you won't need it):

![cdw]({% image_path che-realestate.png %})

Users of Eclipse, IntelliJ IDEA or Visual Studio Code will see a familiar layout: a project/file browser on the left, a code editor on the right, and a terminal at the bottom. You'll use all of these during the course of this workshop, so keep this browser tab open throughout. **If things get weird, you can simply reload the browser tab to refresh the view.**

In the project explorer pane, click on the `Import Projects...` and enter the following:

  * Version Control System: `GIT`
  * URL: `{{GIT_URL}}/userXX/cloud-native-workshop-v2m3-labs.git`(IMPORTANT: replace userXX with your lab user)
  * Check `Import recursively (for multi-module projects)`
  * Name: `cloud-native-workshop-v2m3-labs`

`Tip`: You can find GIT URL when you click on {{GIT_URL}} then login with your credentials. 

![codeready-workspace-import]({% image_path codeready-workspace-import.png %}){:width="700px"}

The projects are imported now into your workspace and is visible in the project explorer.

CodeReady Workspaces is a full featured IDE and provides language specific capabilities for various project types. In order to 
enable these capabilities, let's convert the imported project skeletons to a Maven projects. In the project explorer, right-click on each project and 
then click on `Convert to Project` continuously.

![codeready-workspace-convert]({% image_path codeready-workspace-convert.png %}){:width="500px"}

Choose `Maven` from the project configurations and then click on `Save`.

![codeready-workspace-maven]({% image_path codeready-workspace-maven.png %}){:width="700px"}

Repeat the above for inventory and catalog projects.

> `NOTE`: the Terminal window in CodeReady Workspaces. For the rest of these labs, anytime you need to run a command in a terminal, you can use the CodeReady Workspaces `Terminal` window.

![codeready-workspace-terminal]({% image_path codeready-workspace-terminal.png %})

##### Connnecting to Openshift

When you access OpenShift web console or the other route URL via HTTPS protocol, you will see `Your Connection is not secure` warning message.
Because, OpenShift uses self-certification to create TLS termication route as default. For example, if you're using Firefox, you will see the following screen.

Click on `Advanced > Add Exception...` then, you can access the HTTPS page when you click on `Confirm Security Exception`!!!

![warning]({% image_path browser_warning.png %})