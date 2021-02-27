# KNative on Kubernetes: Serverless without Vendor-Lock-in

Previously I had shared an [article about installing minikube and deploying apps](https://cambazm.medium.com/kubernetes-from-install-to-deploy-minikube-3007f0c1d616). You can also leverage [Microk8s package of Ubuntu](https://ubuntu.com/tutorials/install-a-local-kubernetes-with-microk8s#1-overview) which is easy to work with.

I wanted to share a serverless approach you can leverage on top of Kubernetes which is KNative at this article.

<b>Pre-reqs:</b>
- Download and install an [Ubuntu Server](https://ubuntu.com/download/server) to [Virtualbox](https://www.virtualbox.org/)
- Enable Docker and Microk8s packages while installing Ubuntu Server
- [Configure port forwarding at Virtualbox](https://unix.stackexchange.com/questions/145997/trying-to-ssh-to-local-vm-ubuntu-with-putty/146028#146028) at your host machine
``` 
    Protocol: TCP
    Host IP: 127.0.1.1
    Host Port: 22
    Guest IP: 10.0.2.1
    Guest Port: 22
```
We want to copy/paste commands from host machine to virtual machine so we will install ssh to Ubuntu Server. Execute below command at Ubuntu Server
``` 
sudo apt install ssh
``` 
At the host machine we should be able to access to our virtual machine via ssh

``` 
ssh -p 22 yourusername@127.0.1.
``` 

# Lets start on KNative:

1- If you had enabled microk8s while installing you do not need to install microk8s, you can install or verify by below command:<br />

```
sudo snap install classic microk8s
```
2- Check microk8s status :<br />

```
sudo microk8s.status --wait-ready
```
3- Enable KNative on Microk8s:<br />

```
sudo echo ‘N;’ | microk8s.enable knative
```
For me since I needed to add my account to microk8s group:<br />

```
sudo usermod -a -G microk8s mc
```

or
<br />
```
sudo chown -f -R mc ~/.kube
```
exit from ssh and connect to ssh again, executing below command will work:<br />

```
sudo echo ‘N;’ | microk8s.enable knative
```
It will take quite a while installing relevant stuff<br />

```
istio-1.5.1/
. 
.
.
namespace/istio-system created
.
.
.
```
4- Check out pods of knative serving to verify:<br />

```
microk8s kubectl get pods -n knative-serving
```
(since microk8s has a custom kubectl if you use “kubectl get pods -n knative-serving” command you will get an error like this: “The connection to the server localhost:8080 was refused — did you specify the right host or port?”)

5- Check out pods of knative eventing to verify:<br />

```
microk8s kubectl get pods -n knative-eventing
```
Check out istio pods to verify:<br />

```
microk8s kubectl get pods -n istio-system
```
6- Install KNative operator and related items:<br />

```
microk8s kubectl apply -f https://github.com/knative/operator/releases/download/v0.20.0/operator.yaml
```
Verify operator installation:<br /> 

```
microk8s kubectl get deployment knative-operator
```
7- Install go and hey load generator tool:<br />

```
sudo snap install go --classic
sudo snap install hey
```
8- Install AutoScaler HPA add-on:<br />

```
microk8s kubectl apply -f https://github.com/knative/serving/releases/download/v0.20.0/serving-hpa.yaml
```
9- Download samples:<br />

```
git clone https://github.com/knative/docs knativedocs
```
[These samples](https://github.com/knative/docs/tree/master/docs/serving/samples) could be executed to understand how to deploy and use KNative. Auto-scale sample:<br />

```
cd knativedocs

microk8s kubectl apply --filename docs/serving/autoscaling/autoscale-go/service.yaml

microk8s kubectl get ksvc autoscale-go
```
Sending 30 seconds of traffic maintaining 50 in-flight requests sample command: :<br />

```
hey -z 30s -c 50 \
  "http://autoscale-go.default.1.2.3.4.xip.io?sleep=100&prime=10000&bloat=5" \
  && kubectl get pods
Sample result:

Summary:
  Total:        30.3379 secs
  Slowest:      0.7433 secs
  Fastest:      0.1672 secs
  Average:      0.2778 secs
  Requests/sec: 178.7861

  Total data:   542038 bytes
  Size/request: 99 bytes

Response time histogram:
  0.167 [1]     |
  0.225 [1462]  |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.282 [1303]  |■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.340 [1894]  |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.398 [471]   |■■■■■■■■■■
  0.455 [159]   |■■■
  0.513 [68]    |■
  0.570 [18]    |
  0.628 [14]    |
  0.686 [21]    |
  0.743 [13]    |

Latency distribution:
  10% in 0.1805 secs
  25% in 0.2197 secs
  50% in 0.2801 secs
  75% in 0.3129 secs
  90% in 0.3596 secs
  95% in 0.4020 secs
  99% in 0.5457 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0007 secs, 0.1672 secs, 0.7433 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0000 secs
  req write:    0.0001 secs, 0.0000 secs, 0.0045 secs
  resp wait:    0.2766 secs, 0.1669 secs, 0.6633 secs
  resp read:    0.0002 secs, 0.0000 secs, 0.0065 secs

Status code distribution:
  [200] 5424 responses
  ```
  
# Conclusion
It is quite neat to have your own local serverless architecture. Needless to say, you must select what is right for your business requirements since software is another tool means to an end. If it does not deliver the business value you seek, decrease your costs nor increase your revenue, nothing will make any sense.
<br />
<br />
Pros: 

- You are not locked-in to any vendor, sometimes this becomes priceless due to regulations, ownership of infrastructure and business agility.
- Easy to leverage your existing Kubernetes cluster, this is an add-on to your existing Kubernetes which could assist you.
- If you do not have predictable payloads and need to scale in/out for atomic short-lived functions, KNative might assist you heavily.

However of course there are possible Cons too: 

- Since KNative works on top of Kubernetes and Kubernetes is quite complex to install, configure and maintain, it might become too complex and expensive to maintain if you do not have the right expertise. 
If you have predictable payloads, cost efficient servers and infrastructure then serverless might not make sense for you.
- KNative could be seen as not mature for production loads since it is reasonably new in the market, make sure that you validate it under your production payload before going live.
- You need to add extra monitoring and alerting mechanisms to make sure lots of “serverless functions” act as they suppose to act.
<br />
Check out these too: 
<br />
- AWS Lambda vs Azure Functions vs KNative https://stackshare.io/stackups/aws-lambda-vs-azure-functions-vs-knative
- Ubuntu KNative documentation https://ubuntu.com/blog/getting-started-with-knative-1 
- Microk8s troubleshooting https://microk8s.io/docs/troubleshooting 
- Microk8s kubectl https://microk8s.io/docs/working-with-kubectl 
- Microk8s ingress config https://microk8s.io/docs/addon-ingress
- KNative documentation https://knative.dev/docs/install/ 
- Sample applications on KNative source https://github.com/knative/docs/tree/master/docs/serving/samples
