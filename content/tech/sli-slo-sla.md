---
title: "SLI SLO and SLAs"
summary: "Understanding SLI SLO and SLA and common SLI used for different types of services"
date: 2024-02-14
tags: ["sre","sli","slo","sla","availability"]
author: "Clement Thomas"
---

   In schools, there is some sort of grading systems for students to tell how well they are doing in academics. Say for a test conducted for 100 marks that would be 

|**Mark**| **Grade / Performance Remark** |
|-------|-------|
|< 40 | Fail |
| > 40 and < 95 | Pass |
| >= 95 | Passed with distinction |

We have such a system to know how a student is performing. Similarly in Software world, **SWE** and **SRE** engineers take care of different services and we need a way to tell if we are doing a good job in managing the service.

Let us assume a team in Company C is developing and maintaining a service S. This service S is used by the  customers of Company C. There will be 

* **internal monitoring** - a monitoring tool from with the same region / DC will be monitoring to see if service S is all ok. 

* **external monitoring** - an external monitoring tool like pingdom, catchpoint or newrelic will be used to monitor the service S from different regions around the world.

And whenever there is an alert either from internal monitoring tool / external monitoring system, the oncall engineer will check, try to identify the problem, fix it and there will be RCAs performed to see if this problem can be avoided in future. And this repeats. another alert - remediation - RCA and so on.

How do we tell if our service meets the expectations of our customers or if we are doing a good job in managing the service. We can use SLIs and SLOs for the same. 

The definition of **SLI** as per Google SRE book is

> An SLI is a **service level indicator** — a carefully defined quantitative measure of some aspect of the level of service that is provided.

or 

> Indicator of the level of service.


Mostly used SLIs are

* **request latency** - How long it takes for the service to give back a response
* **error rate** ( fraction of all requests received that errored out ) or **success rate** (number of successful http requests / total http requests )
* **availability** - in the given period of time, how much time was the service usable

* **durability** - how reliably we store data without data loss 
* **system throughput** - no of requests that can be processed per second

Also specific SLIs are selected based on the type of the service. Most common SLIs are

* **SLI for user facing systems**
  - availability
  - latency
  - throughput

* **storage systems**
  - availability
  - latency
  - durability

* **big data systems - data processing pipelines**
  - throughput
  - end-to-end latency

An **SLO** is 

> a target value or range of values for a service level that is measured by an SLI. A natural structure for SLOs is thus SLI ≤ target, or lower bound ≤ SLI ≤ upper bound.

For **availability** SLI, the SLO can be a target value. For eg. **99.99%** availability in a month, which means the service can be down for **4.38** minutes in a month and anymore downtime in a month is considered a violation of SLO. 

For **request latency** SLI again, the SLO can be a target value. For eg. All requests should have latency less than 900ms. Or if we consider that too aggressive, we can also have multiple SLOs as 95% or more requests should complete within 900ms and the rest should complete within 1.5sec. 

An finally **SLAs** are 

> **service level agreements** -  an explicit or implicit contract with your users that includes consequences of meeting (or missing) the SLOs they contain.

So an **SLI** is the key metric or list of metrics  we choose to understand how well we are managing our service and **SLO** defines the acceptable values for those metric(s) and **SLA** is the agreement which defines the penalty we have to pay for not meeting our defined SLOs. 

### References

* https://sre.google/sre-book/service-level-objectives/
* https://sre.google/workbook/implementing-slos/
* https://sre.google/workbook/slo-document/

