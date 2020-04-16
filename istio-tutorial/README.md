# istio-tutorial

# Setup namespace

kubectl create namespace tutorial
  
kubectl config set-context $(kubectl config current-context) --namespace=tutorial


# Clone Repo


#git clone https://github.com/redhat-developer-demos/istio-tutorial

cd istio-tutorial

# Deploy
kubectl apply -f <(istioctl kube-inject -f customer/kubernetes/Deployment.yml) -n tutorial

kubectl create -f customer/kubernetes/Service.yml -n tutorial

kubectl create -f customer/kubernetes/Gateway.yml -n tutorial

kubectl apply -f <(istioctl kube-inject -f preference/kubernetes/Deployment.yml)  -n tutorial

kubectl create -f preference/kubernetes/Service.yml -n tutorial

kubectl apply -f <(istioctl kube-inject -f recommendation/kubernetes/Deployment.yml) -n tutorial

kubectl create -f recommendation/kubernetes/Service.yml -n tutorial

kubectl get pods -w -n tutorial

# Check ingress and get the GATEWAY_URL

kubectl get svc istio-ingressgateway -n istio-system

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')

export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')

export INGRESS_HOST=$(minikube ip)

export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

# Check its running

./scripts/run.sh $GATEWAY_URL/customer

# View the dashboards

istioctl dashboard kiali &

istioctl dashboard jaeger &

kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &

http://localhost:3000/dashboard/db/istio-mesh-dashboard 

# Deploy v2 - to see load balanced traffic across v1 & v2

kubectl apply -f <(istioctl kube-inject -f recommendation/kubernetes/Deployment-v2.yml) -n tutorial

kubectl scale --replicas=2 deployment/recommendation-v2 -n tutorial

kubectl get pods

# Move all traffic to v2

kubectl create -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial

kubectl create -f istiofiles/virtual-service-recommendation-v2.yml -n tutorial

# Move traffic back to v1

kubectl replace -f istiofiles/virtual-service-recommendation-v1.yml -n tutorial

# Back to v1 & v2

kubectl delete -f istiofiles/virtual-service-recommendation-v1.yml -n tutorial

# Routing 90% traffic to v1

kubectl create -f istiofiles/virtual-service-recommendation-v1_and_v2.yml -n tutorial

kubectl replace -f istiofiles/virtual-service-recommendation-v1_and_v2_75_25.yml -n tutorial

# Cleanup

kubectl delete -f istiofiles/virtual-service-recommendation-v1_and_v2_75_25.yml -n tutorial

kubectl delete -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial

# User specific routing

kubectl create -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial

kubectl create -f istiofiles/virtual-service-recommendation-v1.yml -n tutorial

kubectl replace -f istiofiles/virtual-service-safari-recommendation-v2.yml -n tutorial

kubectl get virtualservice -n tutorial

curl -A Safari -l $GATEWAY_URL/customer

# Cleanup Safari rule

kubectl delete -f istiofiles/virtual-service-safari-recommendation-v2.yml -n tutorial

# Mobile users to v2

kubectl create -f istiofiles/virtual-service-mobile-recommendation-v2.yml -n tutorial

curl -A "Mozilla/5.0 (iPhone; U; CPU iPhone OS 4(KHTML, like Gecko) Version/5.0.2 Mobile/8J2 Safari/6533.18.5" $GATEWAY_URL/customer

# Cleanup mobile user

kubectl delete -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial

kubectl delete -f istiofiles/virtual-service-mobile-recommendation-v2.yml -n tutorial

# Clean up
./scripts/clean.sh tutorial
