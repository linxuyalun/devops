# Argus

[![Build Status](http://47.112.23.163:8081/api/badges/linxuyalun/devops/status.svg)](http://47.112.23.163:8081/linxuyalun/devops)
[![LICENSE](https://img.shields.io/badge/license-NPL%20(The%20996%20Prohibited%20License)-blue.svg)](https://github.com/linxuyalun/devops/master/LICENSE)
<a href="https://996.icu"><img src="https://img.shields.io/badge/link-996.icu-red.svg" alt="996.icu"></a>

Argus is the assignment of **Software Innovation and R&D Management**. The assignment consists of two parts, one is the demo of our hypothesis, the other which is the main part is make our application cloud native.

## Team Members

* [JamesCao2048](https://github.com/JamesCao2048)
* [pengy14](https://github.com/pengy14)
* [linxuyalun](https://github.com/linxuyalun)

## Hypothesis

Our hypothesis is to build a price comparison platform. It can trigger an event via email to inform the user of the discount of commodity that they want. Commodities could origin from various shopping sites, such as amazon and jingdong. The basic procedure is as followed:
* login
* search commodities in amazon or jingdong with key words
* add commodities to listening list
* start listening
* send an email when get a discount

### Advantages
* no need to login amazon or jingdong 
* integration of multiple electronic business platforms
* track history prices
* discount notification

### Customer Validation
Customer validation is carrired out through online questionnaires, which consists of two rounds. The first round is designed to validate the necessity of price comparison platform. The second round is designed to validate the specified functions of our system. The [Customer Validation Report](doc/customer_validation.md) presents the process and result in detail.

### Demo

We manage our project by submodule, and here are our [front-end code](https://github.com/pengy14/Argus-front) and [back-end code](https://github.com/JamesCao2048/Argus_Backend).


## Cloud Native

Cloud native technologies empower organizations to build and run scalable applications in modern, dynamic environments such as public, private, and hybrid clouds. Containers, service meshes, microservices, immutable infrastructure, and declarative APIs exemplify this approach.

These techniques enable loosely coupled systems that are resilient, manageable, and observable. Combined with robust automation, they allow engineers to make high-impact changes frequently and predictably with minimal toil.

This [doc](doc/cloud-native.md) gives you a really simple guide to devops by using cloud-native toolkits in practice.The doc is composed of the following parts:

* [Continous Integration & Delivery](doc/cloud-native.md#continous-integration--delivery)
* [Scheduling & Orchestration Overview](doc/cloud-native.md#scheduling--orchestration-overview)
* [Kubernetes in Practice](doc/cloud-native.md#kubernetes-in-practice)

