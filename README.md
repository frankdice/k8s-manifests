# k8s-manifests
Kubernetes manifests for home server items


## Bootstrapping
```
cd cluster-services/fluxcd/bluey/ 
# Install the CRD's and such
kustomize build . | kubectl apply -f -
cd clusters/bluey/cluster-services/
# And the Kustomizations that will manage FluxCD and the Bluey folder's apps
kubectl apply -f fluxcd.yaml
```

Then the manual bit of external-secrets

Then everything else

