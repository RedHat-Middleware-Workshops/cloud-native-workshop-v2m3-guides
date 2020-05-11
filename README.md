The CCN Roadshow(Dev Track) Module 3 Guide 2019
===
This module enables developers to integrate Istio service mesh with exsiting cloud-native application and monitor funtionalities via Kiali, Prometheus, Grafana, and more.
The developers also will learn how to authenticate the cloud-native apps via Red Hat Single Sign-On server of Red Hat Application Runtimes.

Agenda
===
* Getting Started with Service Mesh
* Implementing Continuous Delivery
* Creating Distributed Services
* Service Visualization and Monitoring
* Advanced Service Mesh Development

Lab Instructions on OpenShift
===

Note that if you have installed the lab infra via APB, the lab instructions are already deployed.


_Migration_: the docs have been migrated from markdown to adoc. 
Using pandoc e.g: 
``` 
for i in *.md; do pandoc --standalone --to=asciidoc --output=$i.adoc $i; done;
```