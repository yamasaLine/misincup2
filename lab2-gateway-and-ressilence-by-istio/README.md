## Lab 2 Introduction

In this lab, our mail goal is to learn and practice Gateway and Resilience patterns in Mircoservices, with the help of Istio platform as well as its addons. Again, it would be better for you to prepare some warmup language for labs going smoothly. 

- [ ] Get familiar with [Istio concepts](https://istio.io/latest/docs/concepts/) including ***Traffic Management(Destination rules, Virtual Services, Sidecars, Timeouts, Retries, Circuit Breakers) and Observability***. In this lab we only cover the 2 big topic in Istio, by the limitation of workshop capacity.
- [ ] Learn what gateway pattern and resilience patterns are, and why they are important for complicated microservice architecture.
- [ ] Get to know some basic information about [Prometheus](https://prometheus.io/) and [Kiali](https://kiali.io/).



## Setup your code base

We leverage Github to help on completing our lab.

1. Use git to clone [order service istio chart.](https://github.com/yamasaLine/orderSvcIstioChart)

2. We encourage you to write your own concurrent test client to test resilience behaviors of Istio, but you can refer [this](https://github.com/yamasaLine/concTestClient/blob/main/main.py) if you find it hard to complete. You need:

   + Prepare python 3.8+ runtime in your machine.

   + Modify number in line 33 and 34 to adjust parallelism of the client. Note that thread pool size should be the same as iterable size which past to `pool.map(func, iterable)` as 2nd parameter.
   + Modify url in line 22 and 24 according to your Istio gateway configuration. If you do not modify the gateway port, then this step is not nessesary.



## Lab 2 Steps

In this section we describe detailed steps about how to complete the lab. Unlike lab1, in this lab there will be no much intended uncompleted manifests for you to fill(although still some to do) , but more about trying and playing around gateway and resilience stuff in Istio. Below are the steps for you to go through the lab.

1. Generally you do not need to modify 3 Java code repos from Lab1. Deploy a new `OrderService` K8S deployment in `default` namespace so that the 2 deployments have different labels but are routed by the same K8S service. I added `appver: v0` and  `appver: v1`, just for a reference purpose.

2. Complete all the #TODOs in order service Istio chart repo **besides file `kitchensvc-vs.yaml` and `destrules-kitchen.yaml`** (they are intended for later steps and will produce conflicts for current step). Once upon you finished them, start a new terminal to call `minikube tunnel` so that your Istio Ingress service could have a external IP as 127.0.0.1, then test your gateway! `http://127.0.0.1:80/order/create/aabb`for order service, `http://127.0.0.1:80/ticket/create/112233` for kitchen service, `http://127.0.0.1:80/order/verify?orderId=zzbb` for restaurant service, they should all get correct response if your gateway works as expected. By this step, rethink how Istio helped us to accept external traffic and routes them to proper K8S service, and also the roles which `VirtualService`, `DestinationRule` and `Gateway` are playing.

3. Go to Kiali console, Graph page, and render traffic you just issued. Note to choose on 'Traffic Distribution' on 'Display' dropdown. The following diagram is expected from you side:

   ![system schema](https://imgur.com/yQ5QbIa.png)

   You could see our traffic routing percentage configuration for order service deployments is taking effect. As your request count grows larger, the request portfolio will be closer to 75%:25%.

4. Starting from this step we will try to cover some resilience components inside Istio. Firstly we need to extend `kitchensvc` deploys to 2 different versioned apps.

   1. Extend your kitchensvc deployment to 2 deployments, so that both of them contains: `app: kitchensvc`, one of it contains label `version: v0` and the other contains `version: v1`, and they have different resource name. Remember to modify your deployment both on deploy labels as well as pod templating! Otherwise your change will not take effect.

   2. Install your new `kitchensvc` deployment, check your action by following screenshots. We recommend you to add 1 more deployment(new labeled kitchensvc app) yaml to your restaurant helm chart, for convenience if your environment needs to re-establish on some conditions.![](https://imgur.com/rjYOFgX.png)

      

   ![](https://imgur.com/kAusN2f.png)

5.  Remove or comment out route section for `kitchensvc` in `vs.yaml`, to ensure that we cannot access `kitchensvc` from external gateway anymore.

6. Modify `installKitchensvcResillience` to **true** in `Values.yaml` in ordersvc Istio chart.

7. Now we will setup a ratelimiter for subset `v1` of  `kitchensvc` , and check the traffic status in Kiali. First, Complete all #TODOs in file `kitchensvc-vs.yaml` and `destrules-kitchen.yaml` to satisfy following requirements:

   + Match subsets v0 and v1 to kitchensvc deployments which has corresponding label of `version`, in`destrules-kitchen.yaml`. 
   + Apply a `trafficPolicy` to `v1` subset in `destrules-kitchen.yaml`. Configure it to allow max TCP connections to 1 and connect timeout to 10ms. You may also need to add some configurations for `trafficPolicy.connectionPool.http`, to ensure some traffic to `kitchensvc` v1 will fail due to lacking of available connections in pool. Refer [this](https://istio.io/latest/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings-TCPSettings) for documentation. Hint: Try reducing `http1MaxPendingRequests` and `maxRequestsPerConnection`.
   + `kitchensvc-vs.yaml`should be configured so that 80%  requests from `orderSvc` to  `kitchensvc` go to subset `v1` and the rest go to `v0`.

8. Adjust papalism degree to 30 threads in our concurrent test client,  and start it to  issue traffic to `orderSvc` gateway, then check your Kiali Graph with following diagram:

   ![](https://imgur.com/YHP4A3s.png)

   Generally we could summarize such points by this diagram:

   + Requests to subset `v0` for `kitchensvc` are all green, but subset `v1` is in warning state, also there are some failed requests to a "unknown kitchensvc host(shown as a key icon in diagram)". These facts reflected our configuration: we provided a rate limiter to subset `v1` so potentially some requests will fail if service under high pressure, but there are no special config for `v0` so all its traffic is green.
   +  The requests proportion for subset `v0` and `v1` roughly followed the traffic weight configuration in  `kitchensvc-vs.yaml` virtualService: about 20% of total to `v0` and rest to v1.

9. Based on the chart we built in #7 and #8, we will try circuit breaker in Istio. We are going to:

   1. Adjust  `trafficPolicy` in  `destrules-kitchen.yaml` to yaml root level, so that it will have impact on both subsets `v0` and `v1`.
   2. Add circuit breaker configuration inside  `trafficPolicy` , in `destrules-kitchen.yaml`, to ensure the **circuit will break for 1min once it checked 1 consecutive 5xx errors**.
   3. By configuration of #1 and #2, if we run our concurrent test client again, then there will be pretty chance that the circuit will break due to errors produced by rate limiter connection restriction. Validate whether our expectation happens in Python concurrent test client log.

11. Now modify `destrules-kitchen.yaml`. You need **firstly move  `trafficPolicy` which contains  rate limiter settings to `spec` root level**, and then add `outlierDetection` section in this level,  to enable circuit breaker for both subsets `v0` and `v1`, then adjust it to match requirements described in #9.2.

12. Adjust papalism degree to 30 threads in our concurrent test client,  and start it to  issue traffic to `orderSvc` gateway, then check your resilience configuration with following diagram:

    ![](https://imgur.com/mWPyTqD.png)

​    Also you should be able to find such 2 log entries in Python concurrent client log output:

  ![](https://imgur.com/duB3bvb.png)

![](https://imgur.com/kmq2Y17.png)

Generally we could summarize such points by these diagrams and screenshots:

+ Recall the diagram of expectation for #8, there should be only small proportion of failed requests (about 2%) in total. Although rate limiter almost kept unchanged when comparing with #8, our circuit breaker configuration produced huge impact on overall traffic status. Almost 90% requests failed in total, and **this is because our circuit is too easy to break by given configuration:  will break for 1min if 1 consecutive 5xx error happened**.
+ By the 2 screenshots, we can see that after the circuit really broke, the program received 500 status codes without any 200 OK code for 54s(roughly 1min), which could prove the Istio circuit breaker was working as expected.

13. Congratulations, you reach the end of Lab2!



## Resilience, outside of Labs

In this lab, we learned and leveraged Istio gateways and resilience stuff to complete our experiments, and analyzed traffic when resilience patterns are applied to a distributed microservice running on cloud native infra. We may feel confusing if we just tried them but never know why these means are invented and improved again and again, with the evolution of Mirco Services.

Generally, microservices overturning changed the way complicated software should be: thousands or even millions lines of code compiled into single monolithic applications, running on expensive  hardwares, to multiple, self-maintainable by small squads, distributed deployable microservices running on clusters consisting of cheap hardwares. Even though so many pros introduced, we definitely need to pay prices, on the other side. Inner-OS calls degraded to  RPCs between different hosts, which brings complexity from network communication; RPSes towards services are now counted by millions, and almost 0 traffic down is tolerable, which brings big challenges to micro service architecture...

Resilience patterns are saves for microservices under unpredictable pressure of traffic. Services are always possible to become slow, overloading or totally down, we need means to handle these cases, and avoid situations when some single point failures broadcasting to whole mircoservice and make overall latency and availability metrics worse. In our lab we implemented Circuit Breaker pattern as well as Bulkhead pattern, and:

+ Both of them are to protect our service from request overflooding when service capability goes down or external request volumes increase quickly. 
+ Circuit Breaker pattern can break the circuit if some criteria met, stop all next incoming requests into service which is under some unexpected error situation, for a period of time. During the time engineers can do hotfixes to service since it is protected by circuit breaker.
+ Bulkhead pattern can do rate limiting per service by connection pools. This will protect service workloads under error or high pressure conditions, if needed.

At last we discuss what the advantages Istio holds to traffic management, as well as implementing resilience patterns. Short answer is, it runs as sidecars and hacks 0 application code. In fact it is extremely important to split the resilience stuff out of the service process/workloads to other process/workloads, because when errors occurring under extreme high load, we need to reduce burden from service process/workloads as possible as we can and let them concentrate on providing stable business functionality. Say, Resilience4j for springCloud, is more feature rich than Isito resilience configuration, but it has to run with the service host together, which is not ideal when talking about service loading under high traffic rate. Transparent designation, is the heart of Istio and also is the biggest value it brings to complicated microservices, or, service mesh.



## Lab submission

Generally you need to provide 4 key screenshot for the lab.

1. Checkout a branch from main, using your Epam name.
2. First one is to prove you can call all 3 service endpoints by external gateway. You can use whatever http client you like but `host:port` must be `127.0.0.1:80` and correct responses should be present in screenshot.
3. 2nd one is like the screenshot in #3, to prove your kiali runs successfully and can watch all services and Istio configurations as expected.
4. 3rd one is like the screenshot in #8, to prove your rate limiter works and some traffic to `kitchensvc:v1` would fail.
5. Submit 3 screenshots like #12, to prove your rate limiter and circuit breaker are both working normally, and the concurrent test client could sense the circuit effect.
6. Submit them to folder `lab2-gateway-and-ressilence-by-istio` and push to remote origin.



## Troubleshooting

+ From time to time, you may see this when your microservice pods and Instio ingress had been running for a while(couple days), and you issue traffic by Istio gateway: 

  `upstream connect error or disconnect/reset before headers. reset reason: connection failure, transport failure reason: TLS error: 268435581:SSL routines:OPENSSL_internal:CERTIFICATE_VERIFY_FAILED`

  ​       Basically this is due to [a known issue of Istio](https://github.com/istio/istio/issues/29254). For now recommended workaround would be restarting all microservice pods in `default` namespace and istio-ingress pods in `istio-ingress` namespace. 

+ Remember to always open `minikube tunnel`, only then the Istio ingress service will have an external IP.
