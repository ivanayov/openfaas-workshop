Tuning auto-scaling

Auto-scaling starts at 1 replica and steps up in blocks of 5:

1->5
5->10
10->15
15->20
You can override the minimum and maximum scale of a function through labels.

Add these labels to the deployment if you want to sacle between 2 and 15 replicas.

com.openfaas.scale.min: "2"
com.openfaas.scale.max: "15"
The labels are optional.

Disabling auto-scaling

If you want to disable auto-scaling for a function then set the minimum and maximum scale to the same value i.e. "1".

As an alternative you can also remove AlertManager or scale it to 0 replicas.


----------


Automatically compatible OpenFaaS

The following are fully compatible with any additional back-ends:

API Gateway
Promethes metrics (tracked through API Gateway)
The built-in UI portal (hosted on the API Gateway)
The Function Watchdog and any existing OpenFaaS functions
The CLI
Asynchronous function invocation
Dependent on back-end:

Secrets or environmental variable support
Windows Containers function runtimes (i.e. via W2016 and Docker)
Scaling - dependent on underlying API (available in Docker & Kubernetes)



------------------


Sample function: Node OS Info (nodeinfo)

Grab OS, CPU and other info via a Node.js container using the os module.

If you invoke this method in a while loop or with a load-generator tool then it will auto-scale to 5, 10, 15 and finally 20 replicas due to the load. You will then be able to see the various Docker containers responding with a different Hostname for each request as the work is distributed evenly.

Here is a loop that can be used to invoke the function in a loop to trigger auto-scaling.

while [ true ] ; do curl -X POST http://127.0.0.1:8080/function/func_nodeinfo -d ''; done
Example:

# curl -X POST http://127.0.0.1:8080/function/func_nodeinfo -d ''

Hostname: 9b077a81a489

Platform: linux
Arch: arm
CPU count: 1
Uptime: 776839
To control scaling behaviour you can set a min/max scale value with a label when deploying your function via the CLI or the API:

  labels:
    "com.openfaas.scale.min": "5"
    "com.openfaas.scale.max": "15"


--------------

Your API Gateway will scale functions according to demand by altering the service replica count in the Docker Swarm or Kubernetes API


!!!! Grafana