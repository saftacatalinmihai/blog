---
layout: post
title: Microservice scaling advantage?
excerpt: Is independent scaling of microservices an advantage ?
date: 2022-06-12
updatedDate: 2022-06-12
tags:
  - draft-post
  - microservice
---

Let's say you have a service that runs a number of steps for it's business process. 
One of those steps consumes high CPU - something to do with cryptographic functions let's say.
You run this service on 3 nodes ( you need 3 at least for high availability, no less )
You get concerned that with increasing traffic to this service the CPU starts to go above 89-90%.

What are your options ?
One answer comes from the much praised solution of using microservices and scale them independently.
