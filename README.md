# vault-config-files

----
Contains config files for test installations of HashiCorp Vault on OpenShift.
Below is a reference installation showing how to set up dynamic database credentials for an application accessing information from a Postgresql database.
Credentials are supplied to the application's pod via an injected sidecar container.
The guide assumes an OpenShift server using Kubernetes v1.21+ (i.e. OCP v4.8+) and was tested on OCP v4.8.22.
It is intended as a reference for setup and operation, for more information on Vault-internal concepts please refer to the [documentation](https://www.vaultproject.io/docs/internals).
> :warning: This guide should not be assumed to conform to best practices concerning HashiCorp Vault or cluster security in general.
> Deviations of particular note will be pointed out throughout this document.
> Please consult HashiCorp Vault documentation as well as other appropriate sources before deploying in an environment with heightened security concerns.

## Installing Vault

----
The recommended way of installing Vault on OpenShift is via Helm Chart.
This reference was made using v0.19.0 of the `hashicorp/vault` chart using Helm v3.5.
```console
$ oc login ...
$ helm repo add hashicorp https://helm.releases.hashicorp.com
$ helm repo update
$ helm install vault-v2 hashicorp/vault -f helm-install-values/test-v2-values.yaml 
```
The [config](https://www.vaultproject.io/docs/v1.9.x/platform/k8s/helm/configuration) file `test-v2-values.yaml` contains the most fundamental parameters for the installation:
```yaml
global:
  openshift: true
...
```
Setting `global.openshift` will deploy OpenShift-specific Resources.

```yaml
...
injector:
  enabled: true
  agentImage:
    tag: "1.9.5"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name : shahin-playground
...
```
The [Vault Agent Injector](https://www.vaultproject.io/docs/v1.9.x/platform/k8s/injector) that will inject sidecars into annotated Pods is enabled and configured here.
Version 1.9.5 is used to conform to the server image version (more on that choice later).
The `namespaceSelector` can be used to limit the namespaces where injection can occur.
```yaml
...
server:
  dev:
    enabled: false
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
  image:
    tag: "1.9.5"
  route:
    enabled: true
...
```
The `server` part of the file configures the main Vault servers.
Disabling [dev mode](https://www.vaultproject.io/docs/v1.9.x/concepts/dev-server) is done to give a more realistic view of operating Vault.
This reference makes use of [high availability mode](https://www.vaultproject.io/docs/v1.9.x/concepts/ha) with a [raft algorithm backed](https://raft.github.io/) storage consensus model to show a robust deployment with three replicas.
The container image is fixed to 1.9.5 in order to use the recommended way of dealing with issues concerning newer Kubernetes versions and ServiceAccount authentication with Vault (this is elaborated on further below in the document).
`route` is minimally configured and serves no practical purpose in this example besides showing that a Route could be configured at this point.
```
...
ui:
  enabled: true
```
The last part of the configuration file enables the web UI, which is a helpful tool when starting out with Vault.

> :warning: TLS is disabled by default and not used throughout this document. 
> If it is desired, consider making changes to `global.tlsDisable` as well as other places in the config.

Once the pods are running but not ready, basic Vault setup can continue.
## Initializing Vault

----
Vault starts off in an uninitialized, [sealed](https://www.vaultproject.io/docs/v1.9.x/concepts/seal) state.
In order to progress, it first needs to be initialized on one of its pods.
```console
$ vault operator init
Unseal Key 1: ....
Unseal Key 2: ....
Unseal Key 3: ....
Unseal Key 4: ....
Unseal Key 5: ....

Initial Root Token: s.....
...
```
Both the unseal keys and the root token are required to access Vault, so they should be saved somewhere immediately.
By initializing, the storage backend and encryption are set up.
However, to get through the encryption and access the storage backend Vault still needs to be unsealed by a quorum of unseal keys.
```
$ vault operator unseal $key1
$ vault operator unseal $key2
$ vault operator unseal $key3
```
Since there are three replicas the other two Vault instances also need to be unsealed using a quorum of the same set of unseal keys generated on the first.
However, they need to join the storage setup of the instance where initialization has taken place first.
The following statements on the other two pods will add their Vault instances to the storage network and unseal them.
```console
$ vault operator raft join http://vault-v2-0.vault-v2-internal:8200
$ vault operator unseal $key1
$ vault operator unseal $key2
$ vault operator unseal $key3
```
Note that the address used for joining might differ depending on the name of the pod where `vault operator init` was executed.

> :warning: In a real deployment, some way to [automatically unseal](https://learn.hashicorp.com/collections/vault/auto-unseal) Vaults (necessary due to some kind of outage) should be set up.

Add this point it might be helpful to have the Vault UI to more easily check changes within Vault.
```console
$ oc expose svc vault-v2-ui
```
## Enable Kubernetes Authentication

----
Next, [Kubernetes authentication](https://www.vaultproject.io/docs/v1.9.x/auth/kubernetes) needs to be enabled and configured so that other actors within the cluster can authenticate with Vault.
It makes use of Service Account Tokens to allow access by specific service accounts to specific resources.
Commands that interact with Vault's storage require logging in first.
```console
$ vault login $rootToken
$ vault auth enable kubernetes
```
> :warning: It is heavily recommended to set up an admin user and revoke the root token. 
> This is not covered in this document.

Data within Vault is organized in a path-style manner.
For example, to add configuration to the newly enabled Kubernetes authentication method the data is written to `auth/kubernetes/config`.
```console
$ vault write auth/kubernetes/config \
    kubernetes_host=https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT
```
This particular way of configuring Kubernetes auth is necessary due to changes regarding service account JWTs starting from [Kubernetes v1.21](https://www.vaultproject.io/docs/v1.9.x/auth/kubernetes#kubernetes-1-21).
The recommended way of adapting to those changes is to use the configuration as written above to use the pod's own service account token as the reviewer.

Service account could theoretically authenticate with Vault at this point, although they could not perform any actions since no permissions have been granted yet.
## Setup Database Engine

----
For the next step, a connection with a PostgreSQL database will be set up in order to gain dynamic credentials.
```console
$ vault secrets enable database
```
Just as with authentication, the different ways of storing secrets, handled by [Secret Engines](https://www.vaultproject.io/docs/v1.9.x/secrets), need to be enabled first.
In this case database credentials are required, thus the [database Secret Engine](https://www.vaultproject.io/docs/v1.9.x/secrets/databases) is used.
Once enabled, a config can be written at the path beginning with `database/config/`, just as with authentication.
However, since more than one database connection may be managed through Vault at the same time the separate configurations get a differently named path.
```console
$ vault write database/config/postgresql \
    plugin_name=postgresql-database-plugin \
    allowed_roles="pg-select-public" \
    connection_url="postgresql://{{username}}:{{password}}@postgresql:5432/db?sslmode=disable" \
    username="postgre_u" password="postgre_p" 
```
`plugin_name=postgresql-database-plugin` designates the [database plugin](https://www.vaultproject.io/docs/v1.9.x/secrets/databases/postgresql) that is used.
`allowed_roles` links this connection to a set of roles (note: we will define one in the next step).
These roles define how user credentials are created within the actual database as well as how their privileges are revoked.

> :warning: Make sure the user in the database referenced here by `username` has appropriate permissions to grant permissions on database objects to the user generated in the next step.
```console
$ vault write database/roles/pg-select-public \
    db_name=postgresql \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    revocation_statements="REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM \"{{name}}\"; \
        DROP ROLE \"{{name}}\";" \
    default_ttl="1h" max_ttl="24h" 
```
Roles are written under the path `database/roles/`.
While the database connection config included a whitelist of allowed roles, the role also needs to declare its connection using `db_name`, where that name is the same as the connection config's name in the path.
Dynamic secrets are managed via [leases](https://www.vaultproject.io/docs/v1.9.x/concepts/lease).
A lease defines the amount of time that a token generated by Vault has access to the associated secret.
Should a lease be revoked or run out of time without renewal Vault will cut access to the secret in some way.
`creation_statements`,`revocation_statements` and `renew_statements` are the commands executed in the database, in this case the PostgreSQL database, on lease creation, revocation and renewal respectively.
Despite no explicit `renew_statements` being defined here Vault will update the role's validity in the database automatically.
The lease's default time-to-live and maximum time-to-live are also defined here.
## Setup Service Account

----
The final step within Vault is to tie in a Service Account and give it the correct privileges to be able to access the generated secrets.
Access privileges are handled in Vault via [policies](https://www.vaultproject.io/docs/v1.9.x/concepts/policies).
```console
$ vault policy write pg-select-public-reader - <<EOF
  path "database/creds/pg-select-public" { 
   capabilities = ["read"]
  }
  EOF
```
The policy definition above gives read permission on the credentials generated with `pg-select-public` from the role of the same name.
Note, however, that the path to the generated credentials ends in the same name but is on `database/creds`. 
Every read on `database/creds/pg-select-public` will return a new set of credentials.
Referring to a previously generated set of credentials (e.g. to extend the time-to-live) is done via its lease.
In this example, the Vault sidecar container handles that.
```console
$ vault write auth/kubernetes/role/pg-selector \
    bound_service_account_names=pg-selector \
    bound_service_account_namespaces=shahin-playground \
    policies=pg-select-public-reader \
    ttl=24h
```
In the command above an access role using Kubernetes authentication is defined, called `pg-selector`.
`bound_service_account_names` declares the names of the Kubernetes Service Accounts that can access this role, while `bound_service_account_namespaces` limits the namespaces.
Note that the role's name and Service Account's name being the same is coincidental and not required.
In `policies` the `pg-select-public-reader` policy defined the step before is added to the role, allowing it to access the generated credentials.

With the Service Account set up the final step will be to annotate the test application to mark it as a target for injection.
```console
$ oc create sa pg-selector --namespace shahin-playground
```
## Annotate the Application

----
The application is supplied here without annotations but with a patch to apply them later to see the Agent Injector in action.

```console
$ oc apply -f apps/web-app.yaml --namespace="shahin-playground"
$ oc patch deployment web-deployment --patch-file patches/app-injection-patch.yaml 
```
The patch:
```yaml
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/role: 'pg-selector'
        vault.hashicorp.com/agent-inject-secret-db-creds: 'database/creds/pg-select-public'
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {
          {{ with secret "database/creds/pg-select-public" -}}
            "db_connection": "host=postgresql port=5432 user={{ .Data.username }} password={{ .Data.password }} dbname=db sslmode=disable"
          {{- end }}
          }
        vault.hashicorp.com/service: 'http://vault-v2.shahin-playground.svc:8200'
```
[Annotations coming from Vault](https://www.vaultproject.io/docs/v1.9.x/platform/k8s/injector/annotations) start with the `vault.hashicorp.com` prefix.
`agent-inject` needs to be set for the Mutating Webhook to inject a sidecar into this pod. 
`service` is usually optional, defining which particular Vault server (addressed via its Service) this pod should use.
It becomes relevant if multiple Vault servers exist on the same cluster, especially if the injectors' `namespaceSelector` is not set during installation.
`role` is the name of the role defined under `auth/kubernetes/role/...` to be used.
Note that the Service Account is absent here since it is already declared in `web-app.yaml`.
`agent-inject-secret...` defines two things:
First is the name of the file where the secret will be saved, in this case `db-creds`.
Secondly, the path of the secret to be accessed in Vault, which is of course contingent on the role having been given the correct permissions via policy.
`agent-inject-template...` finally defines the format of the secret in the file, processing it in a way to be digestible by the application.

----
After the patch is applied and the sidecar is injected the secret should be accessible under `/vault/secrets/db-creds` on the pod.