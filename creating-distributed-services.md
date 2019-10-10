## Lab1 - Creating Distributed Services

In this step, we'll install a sample application into the system. This application
is included in Istio itself for demonstrating various aspects of it, but the application
isn't tied exclusively to Istio - it's an ordinary microservice application that could be
installed to any OpenShift instance with or without Istio.

The sample application is called _Bookinfo_, a simple application that displays information about a
book, similar to a single catalog entry of an online book store. Displayed on the page is a
description of the book, book details (ISBN, number of pages, and so on), and a few book reviews.

The BookInfo application is broken into four separate microservices:

* **productpage** - The productpage microservice calls the details and reviews microservices to populate the page.
* **details** - The details microservice contains book information.
* **reviews** - The reviews microservice contains book reviews. It also calls the ratings microservice.
* **ratings** - The ratings microservice contains book ranking information that accompanies a book review.

There are 3 versions of the reviews microservice:

* Version v1 does not call the ratings service.
* Version v2 calls the ratings service, and displays each rating as 1 to 5 black stars.
* Version v3 calls the ratings service, and displays each rating as 1 to 5 red stars.

The end-to-end architecture of the application is shown below.

![Bookinfo Architecture]({% image_path istio_bookinfo.png %})

####1. Deploy Bookinfo Application

---

First, open a new brower with the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"}

![openshift_login]({% image_path openshift_login.png %})

Login using:

* Username: `userXX`
* Password: `r3dh4t1!`

> **NOTE**: Use of self-signed certificates
>
> When you access the OpenShift web console]({{ CONSOLE_URL}}) or other URLs via _HTTPS_ protocol, you will see browser warnings
> like `Your > Connection is not secure` since this workshop uses self-signed certificates (which you should not do in production!).
> For example, if you're using **Chrome**, you will see the following screen.
>
> Click on `Advanced` then, you can access the HTTPS page when you click on `Proceed to...`!!!
>
> ![warning]({% image_path browser_warning.png %})
>
> Other browsers have similar procedures to accept the security exception.

You will see the OpenShift landing page:

![openshift_landing]({% image_path openshift_landing.png %})

> The project displayed in the landing page depends on which labs you will run today. If you will develop `Service Mesh and Identity` then you will see pre-created projects as the above screeenshot.

In order to deploy applicaitons to OpenShift via **oc** tool, we need to copy login command and Login OpenShift cluster:

![codeready-workspace-copy-login-cmd]({% image_path codeready-workspace-oc-login-copy.png %}){:width="700px"}

Then you will redirect to OpenShift Login page again. 

![openshift_login]({% image_path openshift_login.png %})

When you login with your credential, you will see **Display Token** link in the redirected page.

![openshift_login]({% image_path display_token_link.png %})

Click on the link and copy the **oc login** command:

![openshift_login]({% image_path your_token.png %})

Paste it on CodeReady Workspaces Terminal window.

Change the empty **userXX-bookinfo** project via CodeReady Workspaces Terminal and you should replace **userXX** with your username:

`oc project userxx-bookinfo`

Deploy the **Bookinfo application** in the bookinfo project:

`oc apply -f /projects/cloud-native-workshop-v2m3-labs/istio/bookinfo.yaml`

Replace your own gateway URL with _REPLACE WITH YOUR BOOKINFO APP URL_ in bookinfo-gateway.yaml.

 * URL format: userXX-bookinfo-istio-system.{{ROUTE_SUBDOMAIN}}

![gateway]({% image_path bookinfo-gateway.png %})

Set the _REPLACE WITH YOUR BOOKINFO APP URL_ as an environment variable in CodeReady Workspaces Terminal:

`BOOK_URL=_REPLACE WITH YOUR BOOKINFO APP URL_`

Create the _ingress gateway_ for Bookinfo:

`oc apply -f /projects/cloud-native-workshop-v2m3-labs/istio/bookinfo-gateway.yaml`

The application consists of the usual objects like Deployments, Services, and Routes.

As part of the installation, we use Istio to "decorate" the application with additional components (the Envoy Sidecars you read about in the previous step).

Let's wait for our application to finish deploying. Go to the overview page in _userxx BookInfo Service Mesh_ project:

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

Confirm that Bookinfo has been **successfully** deployed via your own _Gateway URL_:

`curl -o /dev/null -s -w "%{http_code}\n" http://$BOOK_URL/productpage`

You should get **200** as a response.

Add default destination rules:

`oc apply -f /projects/cloud-native-workshop-v2m3-labs/istio/destination-rule-all.yaml`

List all available destination rules:

`oc get destinationrules -o yaml`

####2. Access Bookinfo

Open the application in your web browser to make sure if it's working. You will find the URL via running the following command in CodeReady Workspaces Terminal:

`echo http://$BOOK_URL/productpage`

It should look something like:

![Bookinfo App]({% image_path bookinfo.png %})

Reload the page multiple times. The three different versions of the Reviews service show the star ratings differently - _v1_ shows no stars at all, _v2_ shows black stars, and _v3_ shows red stars:

* **v1**: ![no stars]({% image_path stars-none.png %})
* **v2**: ![black stars]({% image_path stars-black.png %})
* **v3**: ![red stars]({% image_path stars-red.png %})

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

Notice that each of the microservices shows **2/2** containers ready for each service (one for the service and one for its sidecar).

Now that we have our application deployed and linked into the Istio service mesh, let's take a look at the immediate value we can get out of it without touching the application code itself!

#####Congratulations!