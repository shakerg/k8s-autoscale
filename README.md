# Prereq: Metric-Server
You must install the metric server in order for data to be collected for use by hpa (Horizonal Pod Autoscaler)
Simply apply all the files in metric-server/1.8+
```
kubectl apply -f metric-server/1.8+/...
```
# Create Horizontal Pod Auotscaler (yaml)
Now that the server is running we can either build an apache-php container and deploy that (See the Dockerfile) or deploy the included yaml which now pulls in ehazlett/docker-demo as a scaling container visualizer. 

# Dockerfile
```
FROM php:5-apache
ADD index.php /var/www/html/index.php
RUN chmod a+rx index.php
```
```
$ docker build -t apache-phpe .
```
The newly created image may be used in the below demonstration or tagged to your DTR and deployed from there.
```
$ docker tag apache-php ${DTR_URL}/demos/apache-php 
$ docker push ${DTR_URL}/demos/apache-php 
```

# Deploy the yaml
Now that the image is prepared, just modify the yaml file for your specific environment
Line #41 for the image location/source 
```
      containers:
      - image: ehazlett/docker-demo
      # - image: dtr.docker.ee/engineering/apache-php:latest
```
Line #63 for the layer7 address
```
spec:
  rules:
  - host: "apache.kube.docker.ee" # layer7 ingress host
```

The yaml file uses the `apache-php` namespace.
```
$ kubectl -n apache-php apply -f apache-php.yaml
```

# Scale test and observe
Auto scaling takes time to both increase and decrease, using kubectl commands we can observe it and also using the ehazlett/docker-demo you may get a visual representation.

We may check the current status of autoscaler by running:
```
$ kubectl get hpa
NAME         REFERENCE                     TARGET    MINPODS   MAXPODS   REPLICAS   AGE
apache-php   Deployment/apache-php/scale   0% / 50%  1         10        1          18s
```

Query the server to generate load, this is using the url per line# 63
```
$ while true; do wget -q -O- http://apache.kube.docker.ee; done
```
Within a minute or so, we should see the higher CPU load by executing:
```
$ kubectl get hpa
NAME         REFERENCE                     TARGET      CURRENT   MINPODS   MAXPODS   REPLICAS   AGE
apache-php   Deployment/apache-php/scale   305% / 50%  305%      1         10        1          3m
```
Here, CPU consumption has increased to 305% of the request. As a result, the deployment was resized to 7 replicas:

```
$ kubectl get deployment apache-php
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
apache-phpe   7         7         7            7           19m
```
Note: It may take a few minutes to stabilize the number of replicas. Since the amount of load is not controlled in any way it may happen that the final number of replicas will differ from this example.
Stop load
We will finish our example by stopping the user load.

In the terminal where we started the while-loop, terminate the load generation by typing Ctrl + C.

Then we will verify the result state (after a minute or so):
```
$ kubectl get hpa
NAME         REFERENCE                     TARGET       MINPODS   MAXPODS   REPLICAS   AGE
apache-php  Deployment/apache-php/scale   0% / 50%     1         10        1          11m
```
```
$ kubectl get deployment apache-php
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
apache-php   1         1         1            1           27m
```


# Create Horizontal Pod Autoscaler (cli)
Now that the server is running, we will create the autoscaler using kubectl autoscale. The following command will create a Horizontal Pod Autoscaler that maintains between 1 and 10 replicas of the Pods controlled by the php-apache deployment we created in the first step of these instructions. Roughly speaking, HPA will increase and decrease the number of replicas (via the deployment) to maintain an average CPU utilization across all Pods of 50% (since each pod requests 200 milli-cores by kubectl run, this means average CPU usage of 100 milli-cores). See here for more details on the algorithm.
```
$ kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
horizontalpodautoscaler.autoscaling/php-apache autoscaled
```
We may check the current status of autoscaler by running:
```
$ kubectl get hpa
NAME         REFERENCE                     TARGET    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   0% / 50%  1         10        1          18s
```
Please note that the current CPU consumption is 0% as we are not sending any requests to the server (the CURRENT column shows the average across all the pods controlled by the corresponding deployment).

Increase load
Now, we will see how the autoscaler reacts to increased load. We will start a container, and send an infinite loop of queries to the php-apache service (please run it in a different terminal):
```
$ kubectl run -i --tty load-generator --image=busybox /bin/sh
Hit enter for command prompt
```
```
$ while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done
```
Within a minute or so, we should see the higher CPU load by executing:
```
$ kubectl get hpa
NAME         REFERENCE                     TARGET      CURRENT   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   305% / 50%  305%      1         10        1          3m
```
Here, CPU consumption has increased to 305% of the request. As a result, the deployment was resized to 7 replicas:

```
$ kubectl get deployment php-apache
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
php-apache   7         7         7            7           19m
```
Note: It may take a few minutes to stabilize the number of replicas. Since the amount of load is not controlled in any way it may happen that the final number of replicas will differ from this example.
Stop load
We will finish our example by stopping the user load.

In the terminal where we created the container with busybox image, terminate the load generation by typing <Ctrl> + C.

Then we will verify the result state (after a minute or so):
```
$ kubectl get hpa
NAME         REFERENCE                     TARGET       MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   0% / 50%     1         10        1          11m
```
```
$ kubectl get deployment php-apache
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
php-apache   1         1         1            1           27m
```
Here CPU utilization dropped to 0, and so HPA autoscaled the number of replicas back down to 1.


# Load Test: 
```while true; do wget -q -O- http://apache.kube.docker.ee; done```