# Consul 1.16 - Envoy extensions - Property Override

# Pre-reqs

1 - Atleast one kubernetes Cluster. 
  
  Important Note if using EKS - there may be an issue with EKS clusters in latest version (1.27) - where the OIDC provider does not get installed. Use this documentation to ensure or deploy if necessary, prior to deploying Consul - https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html

2 - Consul installation on 1.16 
  Helm values here deploy 1.16-dev as I have been deploying pre and post Release candidate for 1.16. Use GA components for everything if it is GA at the time of your testing. 
  
  It may also have additional parameters you dont necessarily need  - in other words a simpler helm values for installing consul would work as long as you are referencing the correct 'image' and 'imageK8S'

  You can install Consul using consul-k8s or helm. There are plenty of helm examples out there, this repo highlights an example of using consul-k8s. 

  Note - If using helm to install, it is better to also use helm to uninstall. 
  
  Consul-k8s is definitely better for uninstall (cleaner in wiping all consul components) so starting with consul-k8s to install is a good idea if you are going to use it to uninstall. 
  
  At the time of publishing this repo, combining helm/consul-k8s is not officially supported. Mix them at your own risk. 

3 - You don't need a Consul Enterprise license for this specific demo but you probably want to get one to test out this and all the other ENT only features in 1.16. See other examples at the end of this readme. 

  To request a 30 day trial license: https://www.hashicorp.com/products/consul/trial

# Deploy Consul on Kubernetes cluster

1. Clone this repo
```
git clone https://github.com/ramramhariram/Consul-Envoyextensions-Propertyoverride.git
```

2. Nagivate to the correct folder. 

```
cd Consul-Envoyextensions-Propertyoverride
```

3. If you have multiple clusters, ensure you are in the current kubernetes cluster context 


4. Add Consul Ent license as a K8s secret, after creating the consul namesapce as well - 

```
export CONSUL_LICENSE=<ADD_YOUR_LICENSE_HERE>
```

```
kubectl create namespace consul 
kubectl create secret generic consul-enterprise-license --from-literal=key=$CONSUL_LICENSE -n consul

```
5 - Ensure you have the correct consul-k8s cli version. Or the correct helm repo if using helm. 
  
  https://developer.hashicorp.com/consul/docs/k8s/installation/install-cli#install-a-previous-version (while it says previous version, you can use these instructions to install newer/RC versions too)

  To install consul - 

  ```
  consul-k8s install --config-file 1.16-servers.yaml -set chart.version=1.2.0-rc1
  ```
  Note - Whether you use consul-k8s or helm, it is always a good practise to set the chart version, especially when working with RC or dev releases.  
  Note - documentation on how to install a specific consul-k8s version of cli (brew may not work in some cases) - 

6 - If you want to ensure the correct version of consul was installed (app version) or if you run into any issues with the install , run the following command to confirm.

  ```
  consul-k8s status
  ```
  

7 - Confirm that consul was installed properly.  You can do kubectl get pods to ensure all the required pods are running. Like this - 

  ```
  kubectl get pods -n consul
  ```

  ```
  NAME                                          READY   STATUS    RESTARTS   AGE
  consul-connect-injector-7dff465dd9-srlj6      1/1     Running   0          2d
  consul-mesh-gateway-6bfd4c9779-29vjn          1/1     Running   0          2d
  consul-mesh-gateway-6bfd4c9779-6vv7h          1/1     Running   0          2d
  consul-mesh-gateway-6bfd4c9779-hkfs5          1/1     Running   0          2d
  consul-server-0                               1/1     Running   0          2d
  consul-server-1                               1/1     Running   0          2d
  consul-server-2                               1/1     Running   0          2d
  consul-webhook-cert-manager-594bd484d-qg74k   1/1     Running   0          2d
  hari@hari-C02FQABCMD6R 1.16 %
  ```

  Note: I installed consul in the Kubernetes 'consul' namespace - this is usually default and recommended in all Hashicorp tutorials. Ensure you use the correct namespace when confirming your kubernetes resources for consul. 

  You could also do kubectl get services to find the consul UI service and login to it. 

  ```
  kubectl get services -n consul
  ```

  ```
  NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)                                                                            AGE
  consul-connect-injector   ClusterIP      172.20.108.249   <none>                                                                    443/TCP                                                                            2d
  consul-dns                ClusterIP      172.20.232.104   <none>                                                                    53/TCP,53/UDP                                                                      2d
  consul-expose-servers     LoadBalancer   172.20.16.28     a3c2511965bce49dab61714510895ed6-573732255.us-east-1.elb.amazonaws.com    8501:31315/TCP,8301:30386/TCP,8300:31611/TCP,8502:31056/TCP                        2d
  consul-mesh-gateway       LoadBalancer   172.20.153.3     a15b798ce94984c0dbcac72a42564a92-902760553.us-east-1.elb.amazonaws.com    443:30773/TCP                                                                      2d
  consul-server             ClusterIP      None             <none>                                                                    8501/TCP,8502/TCP,8301/TCP,8301/UDP,8302/TCP,8302/UDP,8300/TCP,8600/TCP,8600/UDP   2d
  consul-ui                 LoadBalancer   172.20.211.48    a7c20168155c74ebf9f518248487ead4-1227175163.us-east-1.elb.amazonaws.com   443:30515/TCP                                                                      2d
  hari@hari-C02FQABCMD6R 1.16 %
  ```

 Note: use this to get the bootstrap token if you want to see everything in the consul ui - 

 ```
 export CONSUL_HTTP_TOKEN=$(kubectl get --namespace consul secrets/consul-bootstrap-acl-token --template={{.data.token}} | base64 -d)
 ```

 And then to get the actual value of the token for use

 ```
 echo $CONSUL_HTTP_TOKEN 
 ```
 Disclaimer: This is for dev/test environments, not an official recommendation for Production. 

  You should be able to login to your UI with this boostrap token to view everything. Now time to set up a few services for our service mesh deployment. 


# Deploy Hashicups and register it with Consul on Kubernetes

  Any demo application with atleast one upstream/downstream pair will do but we use the hashicups application here to showcase the different options that are possible with propertyoverride and other envoy extensions

  You are free to use any application but you need to make sure you update the propertyoverride.json file accordingly. 

  1 - Clone this repo 

  ```
  git clone https://github.com/hashicorp-education/learn-consul-service-mesh-deploy.git
  ```

  And from that folder, apply just the hashicups configurations as follows - 

  ```
  k apply -f learn-consul-service-mesh-deploy/hashicups
  ```
  The above command should install the application into the default K8s namespace. You can also deploy into a specific namespace if you so choose. 

  2 - Confirm the services are live by running kubectl get services like - 


  ```
  kubectl get pods
  ```

  ```
  hari@hari-C02FQABCMD6R 1.16 % 
  NAME                            READY   STATUS    RESTARTS   AGE
  frontend-f74f5f4d4-69zf7        2/2     Running   0          2d3h
  nginx-7548559b87-6nls9          2/2     Running   0          2d3h
  payments-5d4fdd6c76-xshrr       2/2     Running   0          2d3h
  postgres-58c9cff4d9-5n28p       2/2     Running   0          2d3h
  products-api-7889bb5479-zwkbz   2/2     Running   0          2d3h
  public-api-79f78675f6-2pc4w     3/3     Running   0          2d3h
  hari@hari-C02FQABCMD6R 1.16 
  ```

  Or check all the services like this - 

  ```
  kubectl get services
  ```

  ```
  % 
  NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
  frontend       ClusterIP   172.20.211.247   <none>        3000/TCP   2d3h
  kubernetes     ClusterIP   172.20.0.1       <none>        443/TCP    2d5h
  nginx          ClusterIP   172.20.200.91    <none>        80/TCP     2d3h
  payments       ClusterIP   172.20.156.214   <none>        1800/TCP   2d3h
  postgres       ClusterIP   172.20.124.227   <none>        5432/TCP   2d3h
  products-api   ClusterIP   172.20.159.26    <none>        9090/TCP   2d3h
  public-api     ClusterIP   172.20.107.182   <none>        8080/TCP   2d3h
  hari@hari-C02FQABCMD6R 1.16 %

  ```


# Configure Property override 

  This repo is not meant to explain everyhing about property override on how it works or why we have this feature. For that please refer to our official documentation here - https://developer.hashicorp.com/consul/docs/connect/proxies/envoy-extensions/usage/property-override

  Or talk to a Hashicorp SE 


  That said, here is one of the (easiest and clear) way to see the effects of this extention. 

  1 - Firstly port-forward the Envoy admin interface your service of choice to which you will be making changes. In this case it is the public-API (it has both upstreams as well as downstreams, and multiple of them) and then login to your browser. Like this - 

  ```
  kubectl port-forward public-api-79f78675f6-2pc4w 19000
  ```

  Note - in the above command, you would want to replace the pod name with your service's pod name. 

  2 - From the browser logon to 127.0.0.1:19000 to login to the service's envoy admin interface. 

  Choose the entire config dump (or you can choose to look at specific listeners/clusters depending on your configuration)

  At this point you should just see regular envoy config dump without your changes. 

  NOTE: Curl from the pod is another option to review envoy config dumps but curl is not available on these pods and the browser is easier to view but bottom line you want to look at the envoy config dumps to review changes. 

  3 - Use the following command to apply your property override changes (using the propertyoverride json file is included in this repo or by creating your own accordingly for your application) - 

  ```
  kubectl apply -f propertyoveride.json
  ```

  ```
  servicedefaults.consul.hashicorp.com/public-api created
  hari@hari-C02FQABCMD6R 1.16 %
  ```

  4 - Now 'refresh' the localhost browser, search for the property override changes and they should be there. For example, using the example provided, you should now see a keepalive probe with a value of 1234 for the products_api upstream only. You should also see a 'custom.stats.inbound' for listeners portion. 

  5 - You may delete the propertyoverride file and check that it is gone in the localhost browser again (you have to refresh the browser everytime to grab the new data from envoy)

