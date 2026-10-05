# Single namespace Cosmo Tech platform

## Requirements
* A running Kubernetes cluster, with:
    * Ingress *(this repo includes default manifests working with `Treafik`)*
    * TLS certificate *(this repo includes default manifests working `letsencrypt-prod` secret from `cert-manager`)*
    * Persistent storage solution (Azure disks, Longhorn etc...)
    * Nodes types
        * `Services` with label:
            * `cosmotech.com/tier: services`
        * `Compute` with labels:
            * `cosmotech.com/tier: compute`
            * `cosmotech.com/size: basic`
* Username/password access to the Cosmo Tech images registry (provided by a Cosmo Tech administrator)
* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
* [helm](https://helm.sh/docs/intro/install/)
* [Babylon](https://github.com/Cosmo-Tech/Babylon) (required only once the Kubernetes namespace is ready)
* PowerBi requirements:
    * existing workspace ID
    * Azure client ID & client secret

## Deployment
> You can customize all the manifests according to your needs \
> For Helm Charts & Docker images pulled from `registry.cosmotech.com`, your Cosmo Tech administrator will provide the right VERSION & TAG to use 

* Shortcut variable to reuse in nexts commands
    ```
    NAMESPACE='MY_NAMESPACE'
    ```

* [Optionnal] Create namespace
    > You can also use an existing namespace
    ```
    kubectl create namespace $NAMESPACE
    ```

* Create Cosmo Tech registry secret
    ```
      COSMO_REGISTRY_USERNAME='MY_USERNAME'
    ```
    ```
      COSMO_REGISTRY_PASSWORD='MY_PASSWORD'
    ```
    ```
    kubectl create secret docker-registry registry-cosmotech --docker-username="$COSMO_REGISTRY_USERNAME" --docker-password="$COSMO_REGISTRY_PASSWORD" --docker-server='registry.cosmotech.com'
    ```

* [Optionnal] Create & configure Keycloak
    ```
    KEYCLOAK_CHART_VERSION='xxxxx'
    KEYCLOAK_TAG='xxxxx'
    KEYCLOAK_PSQL_CHART_VERSION='xxxxx'
    KEYCLOAK_PSQL_TAG='xxxxx'
    ```
    * Keycloak database (PostgreSQL)
        > You can also use an existing PostgreSQL instance
        ```
        helm -n $NAMESPACE upgrade --install keycloak-postgresql oci://registry.cosmotech.com/proxy-chainguard-charts/postgresql --values manifests/helm-values-postgresql.yaml --version $KEYCLOAK_PSQL_CHART_VERSION
        ```

    * Keycloak itself
        > You can also use an existing Keycloak instance
        ```
        helm -n $NAMESPACE upgrade --install keycloak-postgresql oci://registry.cosmotech.com/proxy-chainguard-charts/postgresql --values manifests/helm-values-keycloak.yaml --version $KEYCLOAK_CHART_VERSION
        ```

    * Keycloak configuration
        * Create a realm
            * Go on Keycloak > Manage realms > Create realm
                * Realm name = `MY_COSMOTECH_REALM`
                * Enabled = *true*

        > On nexts step, we assume the created realm is selected

        * Create clients
            * For each entry in the table, follow next steps
                | Client                                | Usage                                         | Auth flows
                |---                                    |---                                            |---
                | `cosmotech-client-admin`              | Cosmo Tech Run API (Keycloak admin access)    | `Standard flow`
                | `cosmotech-client-api`                | Cosmo Tech Run API (backend)                  | `Standard flow`
                | `cosmotech-client-web`                | Cosmo Tech Run API (Swagger)                  | `Standard flow`, `Service account roles`
                | `cosmotech-client-babylon`            | Babylon usage                                 | `Standard flow`, `Service account roles`
                | `cosmotech-client-business-webapp`    | Cosmo Tech business webapp                    | `Standard flow`
                * Go to Clients > Create client
                    * General settings
                        * Client ID             = *name*
                        * Name                  = *name*
                        * Always display in UI  = *false*
                    * Capability config
                        * Client authentication         = *true*
                        * Authorization                 = *false*
                        * Authentication flow           = *See **auth flows** section in the table*
                        * Require PKCE                  = *false*
                        * Require DPoP bound tokens     = *false*
                    * Login settings
                        * Root URL = *cluster URL*
                            > example: https://platform.example.com
                        * Home URL = *Cosmo Tech Run API URL suffix*
                            > example: /my_namespace/api
                        * Valid redirect URIs =
                            * `/*`
                            * `https://platform.example.com/my_namespace/api/swagger-ui/oauth2-redirect.html`
                        * Valid post logout redirect URIs = *empty*
                        * Web origins = `+`
                        * Admin URL = *empty*
                * Click on "save"

        * Create client mapper
            * Go to Clients scope > `profile` > Mappers > Add mapper > By configuration > `Group Membership`
                * name                              = `cosmotech-api-groups`
                * Token Claim Name                  = `groups`
                * Full group path                   = *false*
                * Add to ID token                   = *true*
                * Add to access token               = *true*
                * Add to lightweight access token   = *false*
                * Add to userinfo                   = *true*
                * Add to token introspection        = *true*

        * Create roles & groups
            * For each entry in the table, follow next steps
                | Client                | Usage
                |---                    |---
                | `Platform.Admin`      | Full permission
                | `Organization.User`   | Grant authentication, working in pair with objects ACL
                * Go to Realm roles > Create role
                    * Role name = *name*

        * Create users
            * Go to users > Create new user
                * Fill user informations
                * Click on "Create"
                * Set a password (go to the user > Credentials > Set password)
                * Asssign a Cosmo Tech role > (go to the user > Role mapping > Assign role > Realm role)

        > Note: you can connect Keycloak with your own IdP to benefit SSO, sync your users etc...

* Create Cosmo Tech Run API
    > Your Cosmo Tech administrator will provide the right VERSION & TAG to use
    ```
    COSMO_RUN_API_CHART_VERSION='5.3.0'
    COSMO_RUN_API_TAG='5.3.0'
    ```
    ```
    helm -n $NAMESPACE upgrade --install cosmotech-api-$NAMESPACE oci://registry.cosmotech.com/product-charts/cosmotech-api --values manifests/helm-values-cosmotech-run-api.yaml --version $COSMO_RUN_API_CHART_VERSION --set image.tag=$COSMO_RUN_API_TAG
    ```

* Create Cosmo Tech Business Webapp
    > Your Cosmo Tech administrator will provide the right VERSION & TAG to use
    ```
    COSMO_WEBAPP_CHART_VERSION='0.3.0'
    COSMO_WEBAPP_TAG='v7.4.0-vanilla'
    ```
    ```
    helm -n $NAMESPACE upgrade --install cosmotech-business-webapp oci://registry.cosmotech.com/product-charts/cosmotech-business-webapp --values manifests/helm-values-cosmotech-business-webapp.yaml --version $COSMO_WEBAPP_CHART_VERSION --set webapp.functions.image.tag=$COSMO_WEBAPP_TAG --set webapp.server.image.tag=$COSMO_WEBAPP_TAG
    ```



* Create 
    ```
    kubectl
    ```


* Create Cosmo Tech workspace
    ```
    mkdir my_workspace
    ```
    ```
    cd my_workspace
    ```
    ```
    babylon init
    ```
    ```
    babylon apply --exclude webapp project/
    ```

* Get a Cosmo Tech simulator engine
    > Your Cosmo Tech administrator will provide the exact simulator engine image to use
    ```
    echo "$COSMO_REGISTRY_PASSWORD" | docker login registry.cosmotech.com -u $COSMO_REGISTRY_USERNAME
    ```
    ```
    docker pull oci://registry.cosmotech.com/my_simulator_engine:tag
    ```



* Configure Keycloak

Made with :heart: by Cosmo Tech DevOps team
