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

## Only manual part (Besides the FluxCD bootstrap), hopefully...
#### Looking for a secret called `op-credentials`
`kubectl create ns 1password`
`kubectl -n 1password create secret generic op-credentials --from-file=1password-credentials.json=./1password-credentials.json`

#### And the Key to let external-secrets work - It's the Bluey vault token

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

#### Test command:
`curl -X GET -H "Accept: application/json" -H "Authorization: Bearer <TOKEN>" <POD-IP>:8080/v1/vaults`  


Then everything else


## External Access

External access means adding a few annotations and manually creating the CNAME in Cloudflare to DuckDNS. Document this better.



## OS Updates

Disable systemd-resolved and use a normal resolver setup (Adguard - binding to 53 messes this up)
https://github.com/AdguardTeam/AdGuardHome/wiki/Docker#resolved   (I just ripped it completely out)


## Look Into Later
- readarr
- openbooks
- Firefly III
- PhotoPrism
- Invidious