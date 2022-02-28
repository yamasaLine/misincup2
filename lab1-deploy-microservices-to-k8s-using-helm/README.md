## Lab 1 Introduction

In this lab, our mail goal is to deploy a set of microservices using Helm charts. As some background knowledge, you should go for the checklist below so that you will not get confusing when implementing the lab.

- [ ] Get familiar with [Kubernetes concepts](https://kubernetes.io/docs/concepts/) including ***Namespace, Pod, Node, Deployment, Replicaset, label selectors, Service***. You need to understand their roles in a cloud-native microservice deployment.
- [ ] After you got some basic understanding with K8S components, then you could go to [Helm concepts](https://helm.sh/docs/topics/charts/) to see how Helm packages K8S resources into a immutable, convenient, deliverable and deployable Chart, pretty much like the thinking of Docker images.
- [ ] In order to implement the lab smoothly, it is also good for you to know basic [Helm templates](https://helm.sh/docs/chart_template_guide/getting_started/). Basically, a commercial Helm Chart will leverage templates a lot to reduce boilerplate manifest code.
- [ ] In the lab we will implement several Java Spring boot application all the way to running K8S deployments. Generally no Java code is required to implement considering  different teck stacks from all of us, but you need to know a little bit of Spring application.yml configure, to complete a K8S service injection task.



## Setup your code base

We leverage Github to help on completing our lab.

1. Use git to clone [orderService](https://github.com/yamasaLine/orderService.git), [kitchenService](https://github.com/yamasaLine/kitchenService.git) and [restaurantService](https://github.com/yamasaLine/restuaurantService.git), total 3 code repos.
2. Use git to clone helm charts [orderServiceChart](https://github.com/yamasaLine/orderSvcChart.git) and [restaurantServiceChart](https://github.com/yamasaLine/restaurantSvcChart.git), total 2 chart repos.



## Lab 1 Steps

In this section we describe detailed steps about how to complete the lab. The 3 code repos and 2 helm chart repos are carefully designed so that **they lack some key pieces to run successfully**. Below are the steps for you to go through the lab.

1. For 3 Java code projects, generally the only change you have to complete is located in OrderService `application.yaml`. You need to complete `restaurantsvc.host`, `restaurantsvc.port`,  `kitchensvc.host` and  `kitchensvc.port`.  Hint: *consider inject environment variables to them, when they are running as containers in K8S cluster*. We will have more detailed info about the task in later steps.
2. Modify `repository` property in plugin: `dockerfile-maven-plugin`, in `pom.xml` for all 3 Java projects. Use `mvn package dockerfile:push "-Ddockerfile.useMavenSettingsForAuth=true"`to produce docker images per Java projects. You should be able to see them in your Docker hub repositories. **Note: close VPN when you do docker push. VPN will have some influence on Minikube network so docker pushing may fail due to this.**
3. **Optional**: We strongly encourage you to understand how docker images are produced and pushed by previous mvn command. How the image tag is computed? Why we have a step 10 in Lab 0 instructions?
4. Leverage `docker run` to test whether your images could run as expected. Remember the task in step 1? When using `docker run` you can pass environment variables into containers and SpringBoot can sense and configure them into Java code. **Do not hardcode `restaurantsvc.host`, `restaurantsvc.port`,  `kitchensvc.host` and  `kitchensvc.port` in orderService**. Try it out.
5. Test the running images: order service by: `GET http://localhost:8762/order/create/aabb`, kitchen service by: `POST http://localhost:8764/ticket/create/112233` and restaurantService by: `GET http://localhost:8763/order/verify?orderId=zzbb`. 
6. Complete all #TODOs in 2 chart repos. We packaged restaurantService and kitchenService together in a single chart, so that you can see how microservices could be organized together using Helm way. You need to fill in the produced image repos and tags in step #3 into `Values.yaml`. 
7. One important task here is to feed order service chart the kitchen and restaurant K8S service names. Note that we created a `configmap` named 'restaurant-svc-name' in restaurant and kitchen service chart, it contains both restaurant K8S service name and kitchen K8S service name. Fill in correct value in the `configmap` and consider how to reference them in order service chart. Only after that you can call order service interface successfully.
8. Time to deploy! Go to the chart repo root folder, leverage `helm -n k8sns install chartName . --dryrun` to test your chart, and use `helm -n k8sns install chartName . --debug` to do real K8S deploy!  
9. Compare your pods status with following screenshot: ![system schema](https://imgur.com/B9aNTtg.png)
10. Use `kubectl -n default port-forward svc/ordersvc-svc 8762` and then you can call order service API using localhost:8762. Call order service `create order` endpoint and you should get 200 OK response containing simple string: `ordersvc:received:kitchen:received restaurant`. Great! That's all you need to implement in Lab1.
11. (***Optional challenge task!***) In this task you will implement a HA pattern while rolling updat e order service deployment. Basically, after some code change and conf change, your order service deployment needs to tolerate a rolling upgrade while facing concurrent client requests, and no traffic loses are allowed. Below are some detailed steps.

	1. Change your order service code, add graceful shutdown mechanism to it so that order service could handle left requests even readiness probe already turned to off. Suggested graceful shutdown timeout would be 20s-30s.
	2. Prepare a client which is capable to call order service `create order` endpoint concurrently, Python, Java or any suitable language can fit. Make sure at least 8 threads of concurrency could be provided.
	3. Start the client created in #2, and meanwhile leverage `helm upgrade --install` command to do a rolling upgrade of order service chart. You need to modify the `deployment.yaml` anyway, either changing cpu/mem resources or changing docker image tag should work.
	4. In the 1st try, your order service is expected to be not HA, with several http requests got failed status. Hint: you need to modify the `deployment.yaml`again, adding some hooks. Why will some reqs fail? **How K8S services and readiness probes cooperate to achieve service HA**? Is there some time gap before K8S order service really knows elder deployment is not available after rolling upgrade started? If so, how to resolve it? 
	5. After the modification in #4, retest order service HA again, by slightly modifying hardware resources for deployment. This time you should see no request failing. Great! Challenge task done!



## Lab submission

Generally you need to provide 2 key screenshot for the lab.

1. Checkout a branch from main, using your Epam name.

2. First one is like screenshot in Lab steps #9, to prove your all 3 microservices run successfully with all readiness probes ready.

3. Then please provide a screenshot that you can call order service interface using K8S service port-forward, and you can get expected result. Curl or any GUI tools are acceptable.

4. Submit them to folder `lab1-deploy-microservices-to-k8s-using-helm` and push to remote origin.
