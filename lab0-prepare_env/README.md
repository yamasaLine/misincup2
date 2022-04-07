## Environment Setup

Due to some reasons, we designed this K8S incubation program to be performed locally, instead of any cloud vendors. Please follow the steps to prepare your local environment. (Note that, all the steps have been validated on win10, powershell. If you want to complete it in WSL or Linux, steps may vary and some extra configurations are potentially needed)

1. Install minikube([here](https://minikube.sigs.k8s.io/docs/start/) the link). Please follow the steps in the page. and execute `minikube tunnel` after minikube installed

2. Make sure you have docker installed(for Win10 it is docker desktop). Prepare a free, personal docker hub account as your docker registry.(go to https://hub.docker.com/) *Note that it is your free to choose alternative docker registries, but the following steps may vary and we did not test such docker registry types.* After that, do a `docker login` in terminal.

3. We suggest that set *memory to 13GB*(for istio to run smoothly) and *cpu to at least 3 cores* when you configure the Minikube parameters. (specifically, use `minikube config set memory 13312` and `minikube config set cpus 4` before using `minikube start`).

4. Install helm latest version([here](https://helm.sh/docs/intro/install/)) locally and add its path to system PATH variable. You could choose win10 `choco` tool to install it by cmd fashion. Test your installation by `helm version`.

5. (*Optional*) You could deploy a simple app to test your local K8S cluster now!

6. Install Istio, using your local Minikube cluster and Helm. [Here](https://istio.io/latest/docs/setup/install/helm/) are the instructions. Make sure to verify your installation in the end! **Note that step #4 in this instruction: "(Optional) Install an ingress gateway" is a must for our lab, so please do complete it!** Verify your installation with the screenshot below.

   ![](https://imgur.com/PIL7OS1.png)

7. Follow the instructions [here](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/) to enable Istio sidecars in Minikube `default` namespace. It is suggested to validate your actions by installing a simple app `sleep` and describe its pod(use `kubectl -n default describe pod $podid`), you should see Istio sidecar container information in output.

8. Follow instructions [here](https://istio.io/latest/docs/ops/integrations/prometheus/)  to prepare Istio Prometheus addon.

9. Follow instructions [here](https://kiali.io/docs/installation/quick-start/)  to complete Kiali installation. Note step #8 is a prerequisite for this step, otherwise you will face lots of errors in Lab2. Remember to choose Helm install way since we have no istioctl installed. Try accessing the UI once installation successful!

10. (*Optional*) Use `helm repo add` cmd to add your helm repository to your local environment.

11. Install Apache Maven cmd tool. 

12. Add following code snippet to your local maven settings.xml, `servers` tag:

    `\<server\>

       \<id\>docker.io\</id\>

       \<username\>yourdockerhub_username\</username\>

       \<password\>yourdockerhub_pw\</password\>

       \<configuration\>
       \<email\>yourdockerhub_mail\</email\>
       \</configuration\>

    \</server\>`

