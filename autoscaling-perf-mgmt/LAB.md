# Autoscaling & Performance Management

## Goal
The goal of this lab is to deploy a webserver, profile its' performance, and use horizontal autoscaling to achieve a desired P90 response time of 1000ms

## Limitations
* Individual pods in the deployment should not consume more than 20m CPU/20Mi memory
* Horizontal scaling should be used to match demand, not vertical

## Learning Outcomes
By completing this lab, you will be able to:
* Use a `HorizontalPodAutoscaler` to scale a deployment in response to demand
* Use ApacheBench to compare the performance of the deployment with and without autoscaling

## High-Level Steps
* Create lab namespace
* Create a deployment running 1 replica of an Nginx webserver
* Use ApacheBench to profile the single replica deployment
* Create an HPA to autoscale the Nginx deployment
* Use ApacheBench to profile the autoscaling deployment and compare performance to the single replica configuration

## Detailed Steps

### Create the Deployment
1. [1]Create a new namespace to host the resources for this lab:

```bash
kubectl create ns autoscaler
```

2. [2]Generate a skeleton deployment configuration:

```bash
kubectl -n autoscaler create deploy nginx --replicas=1 --image=nginx:alpine --port=80 --dry-run=client -o yaml > deploy.yml
```

3. [3]Edit `deploy.yml`, updating the resources stanza as follows:

```yaml
resources:
  requests:
    cpu: 20m
    memory: 20Mi
  limits:
    cpu: 20m
    memory: 20Mi
```

4. [4]Apply the manifest:

```bash
kubectl apply -f deploy.yml
```

5. [5]Expose the deployment via a `NodePort` service:

```bash
kubectl -n autoscaler expose deploy nginx --type=NodePort
```

6. [6]Make a note of the port used by the nodeport service (`kubectl -n autoscaler get svc nginx`) and the IP of your cluster node. You will need this information in the next step

### Profile the Single-Replica Deployment
7. [7]Install the `apache2-utils` package, which includes ApacheBench:

```bash
sudo apt-get update && sudo apt-get install -y apache2-utils
```

8. [8]Use ApacheBench to profile the performance of the deployment:

```bash
ab -n 5000 -c 100 http://<cluster-node-ip>:<service-port>/
```
This command will execute a total of 5000 requests, 100 at a time, and display information about the response time percentiles. Make a note of the 90th percentile value - when developing the lab the observed value was approx. 2800 ms, your results may differ

```
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.5      0       4
Processing:     5 1928 742.0   1999    4465
Waiting:        1 1819 695.5   1898    3979
Total:          5 1929 741.9   1999    4466

Percentage of the requests served within a certain time (ms)
  50%   1999
  66%   2201
  75%   2458
  80%   2598
  90%   2801
  95%   3100
  98%   3399
  99%   3498
 100%   4466 (longest request)
```

### Configure Autoscaling
Time to use horizontal autoscaling to improve our deployments' performance

9. [9]Autoscaling requires the metrics server be installed, so install using the provided script:

```bash
./install-metrics-server.sh
```

10. [10]Generate a HPA resource manifest using the `kubectl autoscale` command:

```bash
kubectl -n autoscaler autoscale deploy nginx --max=5 --cpu-percent=60 --dry-run=client -o yaml > hpa.yml
```

11. [11]Apply the manifest to create the HPA resource:

```bash
kubectl apply -f hpa.yml
```

12. [12]Wait a few moments for the HPA to begin receiving metrics for the deployment, then repeat the benchmark procedure from before:

```bash
ab -n 5000 -c 100 http://<cluster-node-ip>:<service-port>/
```
Compare the results to those observed without the autoscaler. In the environment where this lab was tested, the 90th percentile response time was 1000 ms. Again, your results may differ

```
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.6      0       7
Processing:     0  461 412.1    400    2554
Waiting:        0  427 387.5    382    2554
Total:          1  461 412.0    400    2554

Percentage of the requests served within a certain time (ms)
  50%    400
  66%    602
  75%    780
  80%    799
  90%   1000
  95%   1243
  98%   1399
  99%   1581
 100%   2554 (longest request)
```

13. [13]Feel free to experiment with the impact of changing the HPA autoscaling parameters, and with changing the number/concurrency of requests sent by ApacheBench

## Review
**Congratulations!** You have successfully deployed and profiled the performance of a webserver, and configured autoscaling to improve said deployment's P90 response time