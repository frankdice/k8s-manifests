Pulled from here:
https://github.com/1Password/connect/tree/main/examples/kubernetes


# Only manual part (Besides the FluxCD bootstrap), hopefully...
Looking for a secret called `op-credentials`
`kubectl create ns 1password`
`kubectl -n 1password create secret generic op-credentials --from-file=1password-credentials.json=./1password-credentials.json`