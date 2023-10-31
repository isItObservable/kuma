# Is it Observable
<p align="center"><img src="/image/logo.png" width="40%" alt="Is It observable Logo" /></p>

## Episode : Kuma
<p align="center"><img src="/image/kuma_logo.jpg" width="40%" alt="Is It observable Logo" /></p>
This repository contains the files utilized during the tutorial presented in the dedicated IsItObservable episode related to Kuma.


This repository showcase the usage of Ambient Mesh with :
* The OpenTelemetry demo
* Dynatrace

We will send all Telemetry data produced by otel-demo to Dynatrace.

## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm


### 1.Create a Google Cloud Platform Project
```shell
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```shell
ZONE=europe-west3-a
NAME=isitobservable-kumamesh
gcloud container clusters create ${NAME} --zone=${ZONE} --machine-type=e2-standard-4 --num-nodes=2
```

## Getting started
### Dynatrace Tenant
#### 1. Dynatrace Tenant - start a trial
If you don't have any Dynatrace tenant , then i suggest to create a trial using the following link : [Dynatrace Trial](https://bit.ly/3KxWDvY)
Once you have your Tenant save the Dynatrace tenant url in the variable `DT_TENANT_URL` (for example : https://dedededfrf.live.dynatrace.com)
```
DT_TENANT_URL=<YOUR TENANT Host>
```

#### 2. Create the Dynatrace API Tokens
Create a Dynatrace token with the following scope ( left menu Acces Token):
* ingest metrics
* ingest OpenTelemetry traces
* ingest logs
<p align="center"><img src="/image/data_ingest.png" width="40%" alt="data token" /></p>
Save the value of the token . We will use it later to store in a k8S secret

```
DATA_INGEST_TOKEN=<YOUR TOKEN VALUE>
```
### Kuma

1. Download kumactl
```shell
curl -L https://kuma.io/installer.sh | VERSION=2.4.3 sh -
```
This command download the latest version of istio ( in our case istio 1.18.2) compatible with our operating system.
2. Add istioctl to you PATH
```shell
cd kuma-2.4.3/bin
```
this directory contains samples with addons . We will refer to it later.
```shell
export PATH=$PWD/bin:$PATH
```

### Clone the Github Repository
```shell
https://github.com/isItObservable/kuma
cd kuma
```

### Deploy most of the components
The application will deploy the entire environment:
```shell
chmod 777 deployment.sh
./deployment.sh   --dthost "${DT_TENANT_URL}" --dttoken "${DATA_INGEST_TOKEN}" 
```
## Tutorial

### 1. Let's Deploy the Observability Stack

#### Logs:
```shell
kubectl apply -f kuma/MeshAccesslog.yaml
```

####  Traces:
```shell
kubectl apply -f kuma/meshtrace.yaml
```

### 3. Parse the logs using the Dynatrace Query Langage

#### Tcp traffic
To parse the logs generated by Kuma with DQL , you can run the following DQL:
```
fetch logs //, scanLimitGBytes: 500, samplingRatio: 1000
| filter matchesValue(log_name, "MeshAccessLog")
| filter contains(content,"took")
| parse content, "LD  SPACE LD SPACE LD:meshname SPACE IPV4 '(' LD:frommeshservice ')' LD IPV4 LD '(' LD:toservice ')' SPACE 'took' SPACE LD:duration 'ms, sent' SPACE LD:bytesent 'bytes, received:' LD:bytereceived SPACE 'bytes'"
```
#### http traffic
To parse the logs generated by Kuma with DQL , you can run the following DQL:
```
fetch logs //, scanLimitGBytes: 500, samplingRatio: 1000
| filter log_name == "MeshAccessLog"
| filter not contains(content,"took")
| parse content ,"LD:intro ']' SPACE LD:meshname SPACE DQS:url SPACE LD:responsecode SPACE LD:Response_flag SPACE LD:Bytereceived SPACE LD:bytesent SPACE LD:duration SPACE LD:durationv SPACE DQS:ip SPACE DQS:useragent SPACE DQS SPACE DQS SPACE DQS:domain SPACE DQS:upstream SPACE DQS:kumaservice SPACE LD"
```

### 2. Let's deploy a Request timeout

```shell
kubectl apply -f kuma/meshtimeout.yaml
```
To identify any request timeout you can do it in 2 different way:
* using logs:
 ```
fetch logs //, scanLimitGBytes: 500, samplingRatio: 1000
  | filter log_name == "MeshAccessLog"
  | filter not contains(content,"took")
  | parse content ,"LD:intro ']' SPACE LD:meshname SPACE DQS:url SPACE LD:responsecode SPACE LD:Response_flag SPACE LD:Bytereceived SPACE LD:bytesent SPACE LD:duration SPACE LD:durationv SPACE DQS:ip SPACE DQS:useragent SPACE DQS SPACE DQS SPACE DQS:domain SPACE DQS:upstream SPACE DQS:kumaservice SPACE LD"
  | filter response_code_detail == "response_timeout"
```
* using metrics:
otherwise you can look at the following metric:
  `envoy_cluster_upstream_rq_timeout.count`
### 3. Let's deploy a Rate limit
```shell
kubectl apply -f kuma/MeshRatelimt.yaml
```

To identify a Rate limits we can do it 2 different way:
* using logs:
```
fetch logs //, scanLimitGBytes: 500, samplingRatio: 1000
  | filter log_name == "MeshAccessLog"
  | filter not contains(content,"took")
  | parse content ,"LD:intro ']' SPACE LD:meshname SPACE DQS:url SPACE LD:responsecode SPACE LD:Response_flag SPACE LD:Bytereceived SPACE LD:bytesent SPACE LD:duration SPACE LD:durationv SPACE DQS:ip SPACE DQS:useragent SPACE DQS SPACE DQS SPACE DQS:domain SPACE DQS:upstream SPACE DQS:kumaservice SPACE LD"
  | filter Response_flag =="RL"
  | filter response_code_detail == "local_rate_limited"
  | summarize count() , by:{bin(timestamp, 30s),dt.kubernetes.workload.name,kumaservice}
```
* using metrics
otherwise you can look at the following metric:
  `envoy_http_local_rate_limit_enabled.count`

### 4. Let's deploy a Traffic Permission
```shell
kubectl apply -f kuma/meshpermission.yaml
```

### 5. Let's deploy a MeshFaultInkection
```shell
kubectl apply -f kuma/MeshFaultINjection.yaml
```

