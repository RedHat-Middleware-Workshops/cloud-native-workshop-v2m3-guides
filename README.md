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

Here is an example Ansible playbook to deploy the lab instruction to your OpenShift cluster manually.

```
- name: Create Guides Module 3
  hosts: localhost
  tasks:
  - import_role:
      name: siamaksade.openshift_workshopper
    vars:
      project_name: "guide-m3"
      workshopper_name: "Cloud-Native Workshop V2 Module-3"
      project_suffix: "-XX"
      workshopper_content_url_prefix: https://raw.githubusercontent.com/RedHat-Middleware-Workshops/cloud-native-workshop-v2m3-guides/master
      workshopper_workshop_urls: https://raw.githubusercontent.com/RedHat-Middleware-Workshops/cloud-native-workshop-v2m3-guides/master/_cloud-native-workshop-module3.yml
      workshopper_env_vars:
        PROJECT_SUFFIX: "-XX"
        COOLSTORE_PROJECT: coolstore{{ project_suffix }}
        OPENSHIFT_CONSOLE_URL: "https://YOUR_OCP_MASTER_URL"
        ECLIPSE_CHE_URL: "http://che-labs-infra.YOUR_OCP_ROUTE_SUBFFIX"
        GIT_URL: "http://gogs-labs-infra.YOUR_OCP_ROUTE_SUBFFIX"
        NEXUS_URL: "http://nexus-labs-infra.YOUR_OCP_ROUTE_SUBFFIX"
        LABS_DOWNLOAD_URL: "http://gogs-labs-infra.YOUR_OCP_ROUTE_SUBFFIX"
         
      openshift_cli: "oc --server https://YOUR_OCP_MASTER_URL"
```
