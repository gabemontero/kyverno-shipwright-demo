# kyverno-shipwright-demo

An initial prototype of using [Kyverno](https://github.com/kyverno/kyverno) to enforce policy on [Shipwright](https://github.com/shipwright-io/build)
and its underlying use of [Tekton](https://github.com/tektoncd/pipeline)

This demo take the Shipwright feature of propagating annotations from the Shipwright BuildStrategy to the Tekton TaskRun generated from the Shipwright BuildRun,
and with the associated Kyverno ClusterPolicy, insures that only the `builder` ServiceAccount is allowed for the underlying Pod.

The try this out, after installing Shipwright, Tekton, and then Kyverno by the install methods promoted at their respective GitHub repositories, first run:
- `oc apply -f ./cluster-build-strategy-with-annotation.yaml`
- `oc apply -f ./kyverno-taskrun-via-shipwright-policy.yaml`
- `oc new-project kyverno-shipwright-demo`
- `REGISTRY_SERVER=https://index.docker.io/v1/ REGISTRY_USER=<your_registry_user> REGISTRY_PASSWORD=<your_registry_password>`
- `oc create secret docker-registry push-secret --docker-server=$REGISTRY_SERVER --docker-username=$REGISTRY_USER --docker-password=$REGISTRY_PASSWORD`
- `oc apply -f ./shipwright-build.yaml`
- `oc create -f ./shipwright-buildrun-allowed-by-kyverno-policy.yaml`

If you then run:
- `oc get br`
- `oc get tr`

You'll see the normal output progression and ultimately completion:

```bash
$ oc get tr
NAME                                            SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
buildpack-nodejs-buildrun-allowed-ws6zq-wfxw6   True        Succeeded   76s         24s
$ oc get br
NAME                                      SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
buildpack-nodejs-buildrun-allowed-ws6zq   True        Succeeded   82s         30s
$ 
```

Conversely, if you run:
- `oc create -f ./shipwright-buildrun-disallowed-by-kyverno-policy.yaml`

Followed by:
- `oc get br`
- `oc get tr`

You'll see that the TaskRun is never created:

```bash
$ oc get br
NAME                                         SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
buildpack-nodejs-buildrun-allowed-ws6zq      True        Succeeded   4m4s        3m12s
buildpack-nodejs-buildrun-disallowed-dckks                                       
$ oc get tr
NAME                                            SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
buildpack-nodejs-buildrun-allowed-ws6zq-wfxw6   True        Succeeded   4m11s       3m19s
$ 
```

And an error log in the Shipwright controller log that looks like:
```
{"level":"error","ts":1619462531.7438319,"logger":"controller-runtime.controller","msg":"Reconciler error","controller":"buildrun-controller","name":"buildpack-nodejs-buildrun-disallowed-dckks","namespace":"kyverno-shipwright-demo","error":"admission webhook \"validate.kyverno.svc\" denied the request: \n\nresource TaskRun/kyverno-shipwright-demo/buildpack-nodejs-buildrun-disallowed-dckks-8g2f9 was blocked due to the following policies\n\ntask-run-inherit-build-strategy-annotation-policy:\n  task-run-inherit-build-strategy-annotation-rule: 'validation error: Only serviceaccount builder can use annotation foo.io from a TaskRun. Rule task-run-inherit-build-strategy-annotation-rule failed at path /spec/serviceAccountName/'\n","stacktrace":"sigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).processNextWorkItem\n\tsigs.k8s.io/controller-runtime@v0.6.1/pkg/internal/controller/controller.go:209\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).worker\n\tsigs.k8s.io/controller-runtime@v0.6.1/pkg/internal/controller/controller.go:188\nk8s.io/apimachinery/pkg/util/wait.BackoffUntil.func1\n\tk8s.io/apimachinery@v0.19.7/pkg/util/wait/wait.go:155\nk8s.io/apimachinery/pkg/util/wait.BackoffUntil\n\tk8s.io/apimachinery@v0.19.7/pkg/util/wait/wait.go:156\nk8s.io/apimachinery/pkg/util/wait.JitterUntil\n\tk8s.io/apimachinery@v0.19.7/pkg/util/wait/wait.go:133\nk8s.io/apimachinery/pkg/util/wait.Until\n\tk8s.io/apimachinery@v0.19.7/pkg/util/wait/wait.go:90"}
```

