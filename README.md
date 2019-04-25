# Cache-Redis-Memcached
Redis and Memcached docker container on an AWS ECS.

* ### Redis dummy data loader

redis-scheduler.yaml will create a CronJob resource in Kubernetes. Documentation: https://cloud.google.com/kubernetes-engine/docs/how-to/cronjobs

    * Invoke every five minutes
    * set Random data in redis cluster using `set $RANDOM $RANDOM`
### redis-scheduler.yml
```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: redis-scheduler
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: redis-scheduler
            image: redis
            command:
            - bash
            - -c
            - redis-cli -h redis-cluster set $RANDOM $RANDOM
          restartPolicy: OnFailure

```

#### To use redis-scheduler.yml, create the resource:

```
kubectl apply -f redis-scheduler.yml
```

#### List your current cronjobs:

```
# kubectl get cronjob
NAME              SCHEDULE    SUSPEND   ACTIVE    LAST SCHEDULE   AGE
example-cronjob   * * * * *   False     0         40s             6m
```

* ## Steps to deploy Memchaed,Redis and Redis dummy data loader

1) Deploy Memcached in EKS

```
kubectl apply -f memcached.yml
```
2) Deploy Redis in EKS as Redis is deplended on memcache so both can deploy on same node.

```
kubectl apply -f redis.yml
```
3) Deploy Redis dummy data loader

```
kubect apply -f schedule.yml
```

## Schedule Redis and memcached server on same node/host in kubernetes.

* Used podAffinity `requiredDuringSchedulingIgnoredDuringExecution` in redis StatefulSet yml to schedule the redis pod in same node where memcached is deployed.

```
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - memcached
              topologyKey: "kubernetes.io/hostname"
  ```
 * Used podAffinity `preferredDuringSchedulingIgnoredDuringExecution` in memcached StatefulSet yml to schedule the memched pod in node where redis is deployed node.
  ```
        affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - redis-cluster
              topologyKey: "kubernetes.io/hostname"
  ```


