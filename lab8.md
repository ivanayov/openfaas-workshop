# Lab 8 - Taking benefit from auto-scaling

OpenFaaS supports auto-scaling. It starts at 1 replica and steps up in blocks of 5:

1->5
5->10
10->15

You can override the minimum and maximum scale of a function through labels. F.e.

```yml
    labels:
      com.openfaas.scale.min: 5
      com.openfaas.scale.max: 15
```

You can disable auto-scaling by setting `com.openfaas.scale.min` and `com.openfaas.scale.max` to 1.

Let's try an example with a memory intensive function.

Create the function with 

```
faas new --lang python3 --prefix <your-docker-username> range-array
```

Now edit `range-array/handler.py` with the following code:

```python
import random

def handle(req):
    d = {}
    i = 0;
    for i in range(0, 100000000):
        d[i] = 'B'*1024

    print("Done")
```

Open `range-array.yml` and disable auto-scaling:

```yml
    labels:
      com.openfaas.scale.min: 5
      com.openfaas.scale.max: 15
```

Invoke the function with 

```bash
$ for i in {1..1000}; do curl -X POST http://127.0.0.1:8080/function/range-array -d '' & done
```

Open another terminal with the docker status log:

```
$ docker stats
CONTAINER ID        NAME                                            CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
adbd9d469494        range-array.1.nres1lqyfy3eyevknhp10gxqe         45.62%              1.193GiB / 1.952GiB   61.14%              245kB / 50.7kB      14.2MB / 54.1MB     168

CONTAINER ID        NAME                                            CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
adbd9d469494        range-array.1.nres1lqyfy3eyevknhp10gxqe         96.47%              1.265GiB / 1.952GiB   64.79%              273kB / 50.9kB      15.8MB / 85.9MB     167
```

You can notice that there is a single container for range-array with high CPU and memory usage.

Now update `range-array.yml` and change the auto-scaling lables:

```yml
    labels:
      com.openfaas.scale.min: 5
      com.openfaas.scale.max: 15
```

Try the same test:

```bash
for i in {1..1000}; do curl -X POST http://127.0.0.1:8080/function/range-array -d '' & done
```

You can now notice that the function is executed in 5 different containers with better results for memory and CPU usage:

```
$ docker stats
CONTAINER ID        NAME                                            CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
fcbce2b8d6f3        range-array.2.snrz143bnupjewj1l46944cd4         0.00%               1.32MiB / 1.952GiB    0.07%               648B / 0B           3.48MB / 0B         5
37a56064732f        range-array.3.krar6c1pu1shkyvywzvi5c13g         1.95%               1.191MiB / 1.952GiB   0.06%               648B / 0B           0B / 0B             5
3cc7cfa5189d        range-array.5.d2l1tlumleok2p7fehr4s4xa5         1.92%               1.242MiB / 1.952GiB   0.06%               648B / 0B           0B / 0B             5
cdc72d2111fa        range-array.1.5fd7d3tlshi7qrdjh1vo2vt47         2.08%               1.102MiB / 1.952GiB   0.06%               578B / 0B           0B / 0B             4
0ee9b4ad5aa8        range-array.4.7mavzfo9j83l0icc71ndcw7xm         2.37%               1.125MiB / 1.952GiB   0.06%               578B / 0B           0B / 0B             5
```

Now return to the [main page](./README.md).