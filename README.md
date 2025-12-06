# oidc-k8s-keycloak

Small guide to integrate Keycloak as an OIDC identity provider for a Kubernetes API server and to wire up cluster Roles/RoleBindings for OIDC groups.

## Quick overview

- Configure a Keycloak client for Kubernetes (OIDC).
- Configure kube-apiserver with OIDC flags and add Keycloak CA.
- Deploy RBAC artifacts (cluster-roles, rolebindings) included in this repo.
- Use the provided client kubeconfig template in `client-cfg/`.

## Keycloak: create client

Create a new client in the realm with these important values:

- Client ID: a GUID (used as the OIDC client id)
- Name: kubernetes (or any friendly name)
- Valid Redirect URIs:
  - http://localhost:18000
  - http://localhost:8000
- Client authentication: Off
- Authorization: Off
- Authentication flow: Standard flow only
- PKCE Method: S256

(Other settings are shown in screenshots in the repo)

## Client side

- See `client-cfg/kubeconfig.yaml` for an example kubeconfig that uses the Keycloak OIDC flow.
- Replace the placeholders (issuer URL, client ID, user, token) with values from your Keycloak client and environment.

## Server side

1. Retrieve Keycloak TLS certificate (PEM):

   ```bash
   openssl s_client -showcerts -connect keycloak.example.com:443 </dev/null 2>/dev/null \
     | openssl x509 -outform PEM > /tmp/keycloak-ca.crt
   ```

2. Copy the CA to the master node and place it where kube-apiserver can read it:

   ```bash
   sudo chown root:root /tmp/keycloak-ca.crt
   sudo chmod 644 /tmp/keycloak-ca.crt
   sudo mv /tmp/keycloak-ca.crt /etc/kubernetes/pki/
   ```

3. Edit the static kube-apiserver manifest (master node):
   - File: `/etc/kubernetes/manifests/kube-apiserver.yaml`
   - Add the following flags to the kube-apiserver command (adjust values):

   ```yaml
   # example snippet inside the kube-apiserver manifest command:
   - --oidc-issuer-url=https://keycloak.example.com/auth/realms/<realm>
   - --oidc-client-id=<OIDC_CLIENT_ID>
   - --oidc-username-claim=preferred_username
   - --oidc-groups-claim=groups
   - --oidc-ca-file=/etc/kubernetes/pki/keycloak-ca.crt
   ```

4. After saving, kubelet will restart kube-apiserver (static pod). Verify API server logs and that OIDC discovery is used.

## RBAC: register cluster roles & bindings

- Apply the provided cluster roles and bindings:

  ```bash
  kubectl apply -f cluster-roles/
  ```

What is provided:
- Administrators: full-cluster access (mapped to an OIDC group)
- Auditors: read-only access (cluster-wide)
- Developers: read-only to all namespaces + full access to dev namespaces

- To give developers full access in additional namespaces, use `rb-dev-oidc-developer.yaml` as a template and apply it in each target namespace.

## Links & references

- Example walkthrough used: https://shivering-isles.com/2024/11/kubernetes-oidc-keycloak
- Official Kubernetes OIDC docs: https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens

## Files of interest in this repo

- client-cfg/ — example kubeconfig(s)
- server-cfg/ — example kube-apiserver flag snippets
- cluster-roles/ — Role / ClusterRole and RoleBinding templates
