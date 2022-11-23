Pulled from here:
https://github.com/1Password/connect/tree/main/examples/kubernetes


# Only manual part (Besides the FluxCD bootstrap), hopefully...
Looking for a secret called `op-credentials`
`kubectl create ns 1password`
`kubectl -n 1password create secret generic op-credentials --from-file=1password-credentials.json=./1password-credentials.json`
And the Key to let external-secrets work - It's the Bluey vault token
`kubectl -n 1password create secret generic bluey-token --from-file=1password-key=./1password-key`

Test command:
`curl -X GET -H "Accept: application/json" -H "Authorization: Bearer <TOKEN>" <POD-IP>:8080/v1/vaults`  