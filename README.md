## How to set up k8s cluster and deploy a sample application with helm
 This repo shows how to spin up very small single master node cluster from scratch on aws with tf, then create helm chart to deploy nginx to k8s. Once app is deployed we then look how to expose it to the world, add secrets that app will use, set is't availability, improve image security, use RBAC so we can query cluster from within the pod, and finally show what todo when k8s api version upgrades happen.

I am using [simplified version of k8s cluster](https://github.com/kenych/terraform_exs/tree/master/k8s) which is based on my [blog post](https://ifritltd.com/2019/06/16/automating-highly-available-kubernetes-cluster-and-external-etcd-setup-with-terraform-and-kubeadm-on-aws/) I wrote few years back on `Automating Highly Available Kubernetes and external ETCD cluster setup with terraform and kubeadm on AWS`, I could have used minikube obviously, but given test nature of this exercise I will create my own cluster.


##  Requirements:
- existing AWS account with a user with enough permissions to create all resources
- terraform
- aws cli
- actual repo to spin up k8s cluster https://github.com/kenych/terraform_exs/tree/master/k8s


##  Setup the test cluster
I described all the steps in the k8s tf repo https://github.com/kenych/terraform_exs/blob/master/k8s/README.md
and they only require terraform access to aws, once configured and applied we can ssh onto the cluster and install the chart.

## The chart for k8s deployment
Each commit resolves one simple task below:
- generate sceleton k8s manifests with helm
- update svc to NodePort type
- add secret for API_KEY
- set availability: evictionStrategy, rollingUpdateStrategy and meaningful replicas
- use nginx-unprivileged user and set custom targetPort as non root can't use 80
- authorise pod to read secrets withing ns
- update api according to newer version

### Generate sceleton k8s manifests with helm

Generate manifests that would be required for docker image to be ready for k8s deployment. Our app is called challengeapp, so simply run:
```
helm create challengeapp
```
### Update svc to NodePort type
We can use NodePort to expose our service to the world. For instance if we were deploying this in aws env with alb, then you could create
- `aws_lb`
- `aws_lb_listener` on 443
- `aws_lb_listener_rule` with some pattern like `/api/*`
- finally froward to `aws_lb_target_group` on nodePort `30080`
- and repeat same for other apps with diff nodePort

### Add secret for API_KEY
In k8s sensitive data like passwords etc is kept in secrets. Out app will be provided with secret api_key during installation. In this example we simply pass it as clear text, in prod we would use:
- aws ssm
- vault 
- jenkins credentials or other secret management tools, line in examples below:

```
helm upgrade --install challengeapp . --set \
 secrets.apiKey=$(aws ssm get-parameters --names "api_key" --query '[Parameters[0].Value]' --output text  --with-decryption --region eu-west-2)
```
```
helm upgrade --install challengeapp . --set \
 secrets.apiKey=$(vault read -field=value secret/api_key/password)
```
```
withCredentials([usernamePassword(credentialsId: 'app_creds', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
	helm upgrade --install challengeapp . --set \
 secrets.apiKey=$(PASSWORD)
}
```
### Set availability: evictionStrategy, rollingUpdateStrategy and meaningful replicas
Next thing we do is ensure high availability of our app, there are at least two issues that matter:
1. what happens when new version is deployed. For that we use maxUnavailable property of the deployment, that is how many pods can be Unavailable during deployment. 
3. What happens when platform admin upgrades the cluster. For this we use PodDisruptionBudget that ensures 
at least 2 instances of the app is always running, this means we can't tolarate anything lower that that, could be used in cases when the number is requried by the quorum (like consul, etcd, or any other fault tolerant app). Normally during draining the node pods will be evicted to a different node, in this case in number of instances is lower draining will fail and you would get error:
```
error when evicting pod "app-2" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.
```

### Use nginx-unprivileged user and set custom targetPort as non root can't use 80
In the fefault nginx image is, the master process is running as root:
```
kubectl exec $(kubectl get pod -l 'app.kubernetes.io/name=challengeapp' --no-headers -o custom-columns=':metadata.name') -- id
uid=0(root) gid=0(root) groups=0(root)
```
It is a security issue, hence we switch to non root image(also official from nginxinc):
```
root@ip-172-31-36-63:~/test-master# kubectl exec $(kubectl get pod -l 'app.kubernetes.io/name=challengeapp' --no-headers -o custom-columns=':metadata.name') -- id
# uid=101(nginx) gid=101(nginx) groups=101(nginx)
```
Because non root user can't spin up a process on port 80, we switch to different port as well.

### RBAC: authorise pod to read secrets withing ns
What is process on the pod wants to access k8s resources? To grant permissions to the clients we need a combo of serviceAccount (who), RoleBinding(glue) and actual role(what), so we can set who can do what on the cluster. The role makes sure clients of the pod can read the secrets in the ns. Example of how it works is shown below in the tests section.

### Update api according to newer version
Surely everyone came accross the annoying error after cluster upgrade:
```
Error: UPGRADE FAILED: unable to recognize "": no matches for kind "PodDisruptionBudget" in version "policy/v1"
```
That is because every version of k8s has different set of api versions, to find out which version is suported with your current cluster version run `api-versions`:
```
kubectl api-versions | grep -i policy
policy/v1beta1
```
it means `PodDisruptionBudget` should be in `policy/v1beta1` for older version of k8s (I used 1.18 deliberately to show this, later with 1.20.x it is updated):
```
kubectl api-versions | grep -i policy
policy/v1
policy/v1beta1
```
As you can see on `v1.23.3` version both `policy/v1` and `policy/v1beta1` are supported, but older api will be deprecated, so we switch to latest. 


## Deploy and test

Once cluster is ready ssh and run the commans below

### Install helm
```
wget https://get.helm.sh/helm-v3.4.1-linux-amd64.tar.gz
tar xvf helm-v3.4.1-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin
rm helm-v3.4.1-linux-amd64.tar.gz
rm -rf linux-amd64
```

### Get the chart
wget https://github.com/kenych/test/archive/master.tar.gz
tar -zxvf  master.tar.gz 
rm master.tar.gz 
cd test-master/

### Install with custom key
helm upgrade --install challengeapp . --set secrets.apiKey=someValue

### Test app is working
```
root@ip-172-31-36-63:~/test-master# curl localhost:30080
 <!DOCTYPE html>
 <html>
 <head>
 <title>Welcome to nginx!</title>
```

### Test API_KEY env var is set

```
root@ip-172-31-36-63:~/test-master# kubectl exec $(kubectl get pod -l 'app.kubernetes.io/name=challengeapp' --no-headers -o custom-columns=':metadata.name') -- printenv API_KEY
#someValue
```

### Test app is non root
```
root@ip-172-31-36-63:~/test-master# kubectl exec $(kubectl get pod -l 'app.kubernetes.io/name=challengeapp' --no-headers -o custom-columns=':metadata.name') -- id
# uid=101(nginx) gid=101(nginx) groups=101(nginx)
```

## Some extras
### Image scan
In prod env we wouldn't just pull random image, even if it is official becasue it may cause security guys to moan, so we would probably pull, scan, and push to our safe place (ecr on aws for instance).

Simple way to scan is shown below, we scan and if scan fails we skip deployment:
```
docker scan --severity=high nginx  > scanres.txt && echo installing && helm upgrade --install challengeapp . --set secrets.apiKey=someValue ||  cat scanres.txt | grep 'Critical severity vulnerability found\|High severity vulnerability found' | tail
# ✗ Critical severity vulnerability found in gnutls28/libgnutls30
# ✗ Critical severity vulnerability found in glibc/libc-bin
# ✗ Critical severity vulnerability found in glibc/libc-bin
# ✗ Critical severity vulnerability found in glibc/libc-bin
# ✗ Critical severity vulnerability found in glibc/libc-bin
# ✗ Critical severity vulnerability found in expat/libexpat1
# ✗ Critical severity vulnerability found in expat/libexpat1
```
If images are in ecr, we can do as below:
```
aws ecr start-image-scan --repository-name nginx --image-id imageTag=v1.16
aws ecr wait image-scan-complete --repository-name nginx --image-id imageTag=v1.16
if [ $(echo $?) -eq 0 ]; then
  SCAN_RESULT=$(aws ecr describe-image-scan-findings --repository-name nginx --image-id imageTag=v1.16 | jq '.imageScanFindings.findingSeverityCounts')
  
  CRITICAL=$(echo $SCAN_RESULT | jq '.CRITICAL')
  HIGH=$(echo $SCAN_RESULT | jq '.HIGH')
  MEDIUM=$(echo $SCAN_RESULT | jq '.MEDIUM')
  
  if [ $CRITICAL != null ] || [ $HIGH != null ]; then
    echo 'ecr image contains CRITICAL issues'
    exit 1
  fi
  helm upgrade --install challengeapp . --set secrets.apiKey=someValue
fi

```
### RBAC

So we know that our pod can read the secrets but other pods can't. Lets prove it. To test let's spin curl pod which will be sending k8s api requests first tith default api token then with the one that service account has generated for our chart (yes, we gonna steal it!).
#### setup curl pod
```
kubectl run curl --image=curlimages/curl \
     -- sh -c "tail -f /dev/null"
```
Install custom token in temp dir:
```
kubectl exec -it curl -- sh -c "echo $(kubectl get secret $(kubectl get secret | grep challengeapp-token | awk '{ print $1 }') -ojson | jq -r '.data."token"'  | base64 -d) > /tmp/token"
```
Get the api endpoint:
```
root@ip-172-31-36-63:~/test-master# kubectl get endpoints | grep kube
# kubernetes     172.31.36.63:6443
````

Send request with default token:
```
/ $ TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
/ $ curl -k -i -s  https://172.31.36.63:6443/api/v1/namespaces/default/secrets/challengeapp --header "Authorization: Bearer $TOKEN"
HTTP/2 403 
..
..
```
Send request with custom token:
```
root@ip-172-31-36-63:~# kubectl exec -it curl -- sh
/ $ TOKEN=$(cat /tmp/token)
/ $ curl -k -i -s  https://172.31.36.63:6443/api/v1/namespaces/default/secrets/challengeapp --header "Authorization: Bearer $TOKEN" | head
HTTP/2 200 
...