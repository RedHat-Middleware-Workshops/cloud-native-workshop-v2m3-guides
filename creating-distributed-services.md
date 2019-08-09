## Lab1 - Creating Distributed Services

In this step, we'll install a sample application into the system. This application
is included in Istio itself for demonstrating various aspects of it, but the application
isn't tied exclusively to Istio - it's an ordinary microservice application that could be
installed to any OpenShift instance with or without Istio.

The sample application is called _Bookinfo_, a simple application that displays information about a
book, similar to a single catalog entry of an online book store. Displayed on the page is a
description of the book, book details (ISBN, number of pages, and so on), and a few book reviews.

The BookInfo application is broken into four separate microservices:

* `productpage` - The productpage microservice calls the details and reviews microservices to populate the page.
* `details` - The details microservice contains book information.
* `reviews` - The reviews microservice contains book reviews. It also calls the ratings microservice.
* `ratings` - The ratings microservice contains book ranking information that accompanies a book review.

There are 3 versions of the reviews microservice:

* Version v1 does not call the ratings service.
* Version v2 calls the ratings service, and displays each rating as 1 to 5 black stars.
* Version v3 calls the ratings service, and displays each rating as 1 to 5 red stars.

The end-to-end architecture of the application is shown below.

![Bookinfo Architecture]({% image_path istio_bookinfo.png %})

#### Setup CodeReady Workspaces for Lab Environment

---

`SKIP this setup guide if you already completed earlier in the other Modules`


Follow these instructions to setup the development environment on CodeReady Workspaces. 

You might be familiar with the Eclipse IDE which is one of the most popular IDEs for Java and other
programming languages. [CodeReady Workspaces](https://access.redhat.com/documentation/en-us/red_hat_codeready_workspaces) is the next-generation Eclipse IDE which is web-based and gives you a full-featured IDE running in the cloud. You have an CodeReady Workspaces instance deployed on your OpenShift cluster
which you will use during these labs.

Go to the [CodeReady Workspaces URL]({{ ECLIPSE_CHE_URL }}) in order to configure your development workspace.

First, you need to register as a user. Register and choose the same username and password as 
your OpenShift credentials.

![codeready-workspace-register]({% image_path codeready-workspace-register.png %})

Log into CodeReady Workspaces with your user. You can now create your workspace based on a stack. A 
stack is a template of workspace configuration. For example, it includes the programming language and tools needed
in your workspace. Stacks make it possible to recreate identical workspaces with all the tools and needed configuration
on-demand. 

For this lab, click on the `Cloud Native Roadshow` stack and then on the `Create` button. 

![codeready-workspace-create-workspace]({% image_path codeready-workspace-create-workspace.png %})

Click on `Open` to open the workspace and then on the `Start` button to start the workspace for use, if it hasn't started automatically.

![codeready-workspace-start-workspace]({% image_path codeready-workspace-start-workspace.png %})

You can click on the left arrow icon to switch to the wide view:

![codeready-workspace-wide]({% image_path codeready-workspace-wide.png %})

It takes a little while for the workspace to be ready. When it's ready, you will see a fully functional 
CodeReady Workspaces IDE running in your browser.

![codeready-workspace-workspace]({% image_path codeready-workspace.png %})

Now you can import the project skeletons into your workspace.

In the project explorer pane, click on the `Import Projects...` and enter the following:

> You can find `GIT URL` when you log in {{GIT_URL}} with your credential(i.e. user1 / openshift).

  * Version Control System: `GIT`
  * URL: `{{GIT_URL}}/userXX/cloud-native-workshop-v2m3-labs.git`
  * Check `Import recursively (for multi-module projects)`
  * Name: `cloud-native-workshop-v2m3-labs`

![codeready-workspace-import]({% image_path codeready-workspace-import.png %}){:width="700px"}

The projects are imported now into your workspace and is visible in the project explorer.

CodeReady Workspaces is a full featured IDE and provides language specific capabilities for various project types. In order to 
enable these capabilities, let's convert the imported project skeletons to a Maven projects. In the project explorer, right-click on each project and then click on `Convert to Project` continuously.

![codeready-workspace-convert]({% image_path codeready-workspace-convert.png %}){:width="500px"}

Choose `Maven` from the project configurations and then click on `Save`.

![codeready-workspace-maven]({% image_path codeready-workspace-maven.png %}){:width="700px"}

Repeat the above for inventory and catalog projects.

> `NOTE`: the `Terminal` window in CodeReady Workspaces. For the rest of these labs, anytime you need to run 
a command in a terminal, you can use the CodeReady Workspaces `Terminal` window.

![codeready-workspace-terminal]({% image_path codeready-workspace-terminal.png %})

####1. Deploy Bookinfo Application

---

We'll use the CLI to deploy the components for our monolith. To deploy the monolith template using the CLI, execute the following commands via CodeReady Workspaces `Terminal` window:

Copy login command and Login OpenShift cluster:

![codeready-workspace-copy-login-cmd]({% image_path codeready-workspace-oc-login-copy.png %}){:width="700px"}

Then you will redirect to OpenShift Login page again. 

![openshift_login]({% image_path openshift_login.png %})

When you login with your credential, you will see `Display Token` link in the redirected page.

![openshift_login]({% image_path display_token_link.png %})

Click on the link and copy the `oc login` command:

![openshift_login]({% image_path your_token.png %})

Paste it on CodeReady Workspaces `Terminal` window.

Change the empty `userXX-bookinfo` project via CodeReady Workspaces `Terminal` and you should replace `userxx` with your username:

`oc project userxx-bookinfo`

Deploy the `Bookinfo application` in the bookinfo project:

`oc apply -f /projects/cloud-native-workshop-v2m3-labs/istio/bookinfo.yaml`

Replace your onw gateway URL with `REPLACE WITH YOUR BOOKINFO APP URL` in `bookinfo-gateway.yaml`.

 * URL format: `userXX-bookinfo-istio-system.<ROUTE SUBFFIX>`
 * URL example: `user1-bookinfo-istio-system.apps.seoul-bfcf.openshiftworkshop.com`

![gateway]({% image_path bookinfo-gateway.png %})

Create the `ingress gateway` for Bookinfo:

`oc apply -f /projects/cloud-native-workshop-v2m3-labs/istio/bookinfo-gateway.yaml`

The application consists of the usual objects like Deployments, Services, and Routes.

As part of the installation, we use Istio to "decorate" the application with additional
components (the Envoy Sidecars you read about in the previous step).

Let's wait for our application to finish deploying. Go to the overview page in `userxx BookInfo Service Mesh` project:

![bookinfo]({% image_path bookinfo-deployed.png %})

Or you can execute the following commands to wait for the deployment to complete and result `successfully rolled out`:

~~~shell
oc rollout status -w deployment/productpage-v1 && \
 oc rollout status -w deployment/reviews-v1 && \
 oc rollout status -w deployment/reviews-v2 && \
 oc rollout status -w deployment/reviews-v3 && \
 oc rollout status -w deployment/details-v1 && \
 oc rollout status -w deployment/ratings-v1
~~~

Confirm that Bookinfo has been `successfully` deployed via your own `Gateway URL`:

`curl -o /dev/null -s -w "%{http_code}\n" http://YOUR_ISTIO_GATEWAY_URL/productpage`

You should get `200` as a response.

Add default destination rules:

`oc apply -f /projects/cloud-native-workshop-v2m3-labs/istio/destination-rule-all.yaml`

List all available destination rules:

`oc get destinationrules -o yaml`

####2. Access Bookinfo

Open the application in your browser to make sure it's working:

* Bookinfo Application running with your own `Gateway URL` at 

`http://YOUR_ISTIO_GATEWAY_URL/productpage`

It should look something like:

![Bookinfo App]({% image_path bookinfo.png %})

Reload the page multiple times. The three different versions of the Reviews service
show the star ratings differently - `v1` shows no stars at all, `v2` shows black stars,
and `v3` shows red stars:

* `v1`: ![no stars]({% image_path stars-none.png %})
* `v2`: ![black stars]({% image_path stars-black.png %})
* `v3`: ![red stars]({% image_path stars-red.png %})

That's because there are 3 versions of reviews deployment for our reviews service. Istio’s
load-balancer is using a _round-robin_ algorithm to iterate through the 3 instances of this service.

You should now have your OpenShift Pods running and have an Envoy sidecar in each of them
alongside the microservice. The microservices are productpage, details, ratings, and
reviews. Note that you'll have three versions of the reviews microservice:

`oc get pods --selector app=reviews`

~~~shell
NAME                          READY   STATUS    RESTARTS   AGE
reviews-v1-7754bbd88-dm4s5    2/2     Running   0          12m
reviews-v2-69fd995884-qpddl   2/2     Running   0          12m
reviews-v3-5f9d5bbd8-sz29k    2/2     Running   0          12m
~~~

Notice that each of the microservices shows `2/2` containers ready for each service (one for the service and one for its
sidecar).

Now that we have our application deployed and linked into the Istio service mesh, let's take a look at the
immediate value we can get out of it without touching the application code itself!

#####Congratulations!