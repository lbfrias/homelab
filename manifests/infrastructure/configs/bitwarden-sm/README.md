# Bitwarden Secrets Manager - Auth Token Setup

The operator needs a machine account access token to authenticate.
This secret must be created MANUALLY before BitwardenSecret resources work.

1. In Bitwarden Secrets Manager:
   - Create a Machine Account for your homelab
   - Generate an access token for the machine account
   - Note your Organization ID (Settings > Organization info)

2. Create the auth token secret:

   kubectl create secret generic bw-auth-token \
     -n <NAMESPACE> \
     --from-literal=token="<YOUR_ACCESS_TOKEN>"

Create this secret in each namespace where you need BitwardenSecret resources.
Or create it in sm-operator-system and reference it from any namespace.
