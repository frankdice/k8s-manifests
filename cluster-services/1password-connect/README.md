Pulled from here:
https://github.com/1Password/connect/tree/main/examples/kubernetes


# Only manual part (Besides the FluxCD bootstrap), hopefully...
Looking for a secret called `op-credentials`
`kubectl create ns 1password`
`kubectl -n 1password create secret generic op-credentials --from-file=1password-credentials.json=./1password-credentials.json`
And the Key to let external-secrets work - It's the Bluey vault token

```
apiVersion: v1
kind: Secret
metadata:
  name: bluey-token
  namespace: 1password
type: Opaque
stringData:
  token: <TOKEN>
```

Test command:
`curl -X GET -H "Accept: application/json" -H "Authorization: Bearer <TOKEN>" <POD-IP>:8080/v1/vaults`  




{"level":"error","ts":1669224804.6108053,"msg":"Reconciler error","controller":"clustersecretstore","controllerGroup":"external-secrets.io","controllerKind":"ClusterSecretStore","ClusterSecretStore":{"name":"bluey"},"namespace":"","name":"bluey","reconcileID":"5c2ae0c2-a5c4-44d6-a85d-5683684307f3","error":"could not validate provider: Get \"http://op-connect.1password:8080/v1/vaults?filter=title+eq+%22Bluey%22\": net/http: invalid header field value for \"Authorization\"",

"stacktrace":"sigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).reconcileHandler
\n\t/home/runner/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.13.0/pkg/internal/controller/controller.go:326
\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).processNextWorkItem
\n\t/home/runner/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.13.0/pkg/internal/controller/controller.go:273
\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).Start.func2.2
\n\t/home/runner/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.13.0/pkg/internal/controller/controller.go:234"}