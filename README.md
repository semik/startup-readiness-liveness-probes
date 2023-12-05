# Startup, Liveness, and Readiness Probes demo

This is my learning project I used for exploration of Startup, Liveness, and Readiness Probes in K8s. It is derived from from [Guided Exercise: Liveness, Readiness, and Startup Probes](https://kubebyexample.com/learning-paths/application-development-kubernetes/lesson-4-customize-deployments-application-3) by [kubebyexample.com](https://kubebyexample.com/).

I've created my own Helm chart to deploy and uninstall all needed components quickly. It depends on simple application from image [do100-probes](https://quay.io/repository/redhattraining/do100-probes?tab=tags&tag=latest) it offers three URI:
  * '/' "an application" for a user which returns text **Hello! This is the index page for the app.**
  * '/ready' which return HTTP 503 for first 30seconds and HTTP 200 later
  * '/healthz' which return health status based on following
  * '/flip' allows to disable and reactivate readines and same for health status, see [source code](https://github.com/RedHatTraining/DO100-apps/blob/main/probes/app.js#L47C13-L47C20)

## Succesfull startup

### Probes definition
```
startupProbe:
  failureThreshold: 3
  httpGet:
    path: /startup
    port: 8080
    scheme: HTTP
  initialDelaySeconds: 20
  periodSeconds: 10
  successThreshold: 1
  timeoutSeconds: 2
readinessProbe:
  failureThreshold: 3
  httpGet:
    path: /ready
    port: 8080
    scheme: HTTP
  initialDelaySeconds: 20
  periodSeconds: 10
  successThreshold: 1
  timeoutSeconds: 2
livenessProbe:
  failureThreshold: 3
  httpGet:
    path: /healthz
    port: 8080
    scheme: HTTP
  initialDelaySeconds: 5
  periodSeconds: 10
  successThreshold: 1
  timeoutSeconds: 2
```
Logs from application:
```
> probes@1.0.0 start
> node app.js

nodejs server running on http://0.0.0.0:8080
19.11: ping /startup => pong [notready]
29.10: ping /startup => pong [notready]
39.10: ping /startup => pong [ready]
39.11: ping /ready => pong [ready]
40.25: serving user request
41.49: serving user request
```

The first time K8s tries the `startupProbe` is after `startupProbe.initialDelaySeconds=20`. Slightly lower value 19.11 than 20 is due a container startup overhead, it takes 0.89s to actualy reach a moment when app reacheches [startup point and is able to record it's startup time](https://github.com/semik/DO100-apps/blob/main/probes/app.js#L91).

Another thing worth mentioning is that readinessProbe.initialDelaySeconds=20 is counted since container startup, **not after `startupProbe` finishes** as I thought at first time. You can see that the first moment where `/ready` mentioned is at 39.11s just after `startupProbe` sucesfully finished.


### App logs:



## Always falling to startup

```
startupProbe:
  failureThreshold: 3
  httpGet:
    path: /ready
    port: 8080
    scheme: HTTP
  initialDelaySeconds: 2
  periodSeconds: 10
  successThreshold: 1
  timeoutSeconds: 2
```

Above startupProbe will not allow application work. The application need 30s to respond to `/ready` OK. There is `initialDelaySeconds: 2`  + ( failureThreshold: 3 * periodSeconds: 10 ) = 32s` which is more than 

```
$ kubectl get events 
LAST SEEN   TYPE      REASON              OBJECT                             MESSAGE
5m13s       Normal    Scheduled           pod/srl-probes-76b8468567-8pztg    Successfully assigned demo/srl-probes-76b8468567-8pztg to node3
3m42s       Normal    Pulled              pod/srl-probes-76b8468567-8pztg    Container image "quay.io/redhattraining/do100-probes:latest" already present on machine
3m42s       Normal    Created             pod/srl-probes-76b8468567-8pztg    Created container srl-probes
3m42s       Normal    Started             pod/srl-probes-76b8468567-8pztg    Started container srl-probes
3m32s       Warning   Unhealthy           pod/srl-probes-76b8468567-8pztg    Startup probe failed: HTTP probe failed with statuscode: 503
3m42s       Normal    Killing             pod/srl-probes-76b8468567-8pztg    Container srl-probes failed startup probe, will be restarted
4s          Warning   BackOff             pod/srl-probes-76b8468567-8pztg    Back-off restarting failed container
5m13s       Normal    SuccessfulCreate    replicaset/srl-probes-76b8468567   Created pod: srl-probes-76b8468567-8pztg
5m13s       Normal    ScalingReplicaSet   deployment/srl-probes              Scaled up replica set srl-probes-76b8468567 to 1
4m37s       Normal    Sync                ingress/srl-probes                 Scheduled for sync
4m37s       Normal    Sync                ingress/srl-probes                 Scheduled for sync
4m37s       Normal    Sync                ingress/srl-probes                 Scheduled for sync
```
```
$ kubectl get events -A
NAMESPACE   LAST SEEN   TYPE      REASON      OBJECT                            MESSAGE
demo        18m         Warning   Unhealthy   pod/srl-probes-76b8468567-8pztg   Startup probe failed: HTTP probe failed with statuscode: 503
demo        4m3s        Warning   BackOff     pod/srl-probes-76b8468567-8pztg   Back-off restarting failed container
```

```
NAME                              READY   STATUS             RESTARTS        AGE
pod/srl-probes-76b8468567-8pztg   0/1     CrashLoopBackOff   141 (80s ago)   6h54m
```


