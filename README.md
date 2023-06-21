# gke-auto-inject
Quick configuration for testing Datadog Auto-Injection with GKE using java app.  
This is used to demonstrate the APM auto-injection capabilities of Datadog in a Kubernetes (GKE) environment
Note - You can find the full source [here](https://github.com/smazzone/springblog-1/tree/main/springback)

## Creating the GKE environment
This assumes you have access to a Google cloud account with the appropriate rights

### Create the network and set firewall rules
1. Create the network - this creates a simple regional network

`gcloud compute networks create <network-name> --project=<project-id> --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional`

2. Create the subnet - set the subnet range as desired

`gcloud compute networks subnets create <subnet-name> --project=<project-id> --range=10.10.0.0/16 --stack-type=IPV4_ONLY --network=<network-name> --region=REGION"`

3. Create the GKE cluster in the region/zone desired

`gcloud beta container --project "<project-id>" clusters create "<cluster-name>" --zone "us-central1-c" \
--no-enable-basic-auth --cluster-version "1.25.8-gke.1000" --release-channel "regular" --machine-type "e2-medium" --image-type "COS_CONTAINERD" \
--disk-type "pd-balanced" --disk-size "100" --metadata disable-legacy-endpoints=true \
--scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write",\
"https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol",\
"https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
--num-nodes "3" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/<project-id>/global/networks/<network-name>" \
--subnetwork "projects/<project-id>/regions/us-central1/subnetworks/<network-name>" --no-enable-intra-node-visibility \
--default-max-pods-per-node "110" --no-enable-master-authorized-networks --tags "<cluster-tag>" \
--addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair \
--max-surge-upgrade 1 --max-unavailable-upgrade 0 --no-enable-managed-prometheus --enable-shielded-nodes --node-locations "us-central1-c"`

4. Create the firewall rules for the subnet and restrict the source to an individual IP

`gcloud compute --project=<project-ID> firewall-rules create <rule-name> \
--direction=INGRESS --priority=1000 --network=<network-name> --action=ALLOW \
--rules=tcp:22,tcp:80,tcp:8080,tcp:8088 --source-ranges=<your IP Addr/32 --target-tags=<cluster-tag>'

## Deploy the Datadog Helm Chart
Install Helm as per the instructions [Datadog Helm](https://docs.datadoghq.com/containers/kubernetes/installation/?tab=helm)
The attached datadog-values.yaml file has the following configured:
- APM enabled
- APM trace collection via Unix Domain Sockets (can also configure for TCP)
- Auto-injection using the AdmissionController in the Datadog Cluster Agent
- Log collection enabled
- Network Performance Monitoring
- Site is set to us1.datadoghq.com

You will need to set the following:
- Datadog API Key
- Cluster Name
- Site (if your Datadog instance is not located in AWS - i.e., US1)

Sample helm deployment:
`helm install ddagent -f datadog-values.yaml --set datadog.apiKey=<your API Key> --set datadog.clusterName=<clustername> --datadog.site=us5.datadoghq.com datadog/datadog`

## Deploy the pods and related infrastructure
This will install the following:
- Frontend springboot framework
- Backend framework
- Load-Balancer service

Sample deployment command:
`kubectl apply -f depl-with-lib-inj.yaml`

## Generating some traces
Once the infrastructure is up, you can generate some traces by using the following:
`curl <exteral-IP-of-load-balancer>:8080/upstream`


