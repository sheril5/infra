# Tekton Flow For Capten
## Tekton CI/CD Pipeline
1. Login to the capten ui page by clicking on this link https://alpha.intelops.app/login
   
2. Onboarding git project in to capten
   
   * select the `add git repo` from the **git** section
   * add the git repo url,access token and label for the customer repo (label is tekton) and the tekton ci/cd repo (label is IntelopsCi)
3. Onboarding container registry in to capten

   * select `add container registry` from **container registry** section
   * add  the registry url,username,access token and label to which the built image needs to be pushed (labels is "tekton")
# Configuring Tekton
## Configuring Capten Tekton Plugin 
   Go to the *capten-->platform engineering* ,select on the tekton setup and then select the `sync` option under the  **configure** section and this will configure the tekton and the neccessary floders will be created in the customer's repo
   
# Pre-requisite For Tekton CI/CD Pipeline Creation

* Create a namepace eg `tekton-pipelines`

* Create a secretstore from the yaml given below.Replace the server with the url which can be obtained from the **kubectl** command given below.

  ```bash
  kubectl get ingress -n capten
  ```
  
          apiVersion: external-secrets.io/v1beta1
          kind: ClusterSecretStore
          metadata:
            name: vault-root-store
          spec:
            provider:
              vault:
                server: <"replace with the ingress host obtained from above command">
                path: "secret"
                version: "v2"
                auth:
                  tokenSecretRef:
                    name: <"secret name of the vault-root-token in tekton namespace">
                    key: "token"
                    namespace: external-secrets

* Git secret
 
 Go to *onboarding-->git* under the respective git project you can see the path of the vault where the credentials of git is stored.copy the path and add it to the path in the external secret yaml as given below

 You must properly annotate the external-secret to specify the domains for which Tekton can use the credentials.

 A credential annotation key must begin with tekton.dev/git- or tekton.dev/docker- and its value is the URL of the host for which you want Tekton to use that credential.

    apiVersion: external-secrets.io/v1beta1
    kind: ExternalSecret
    metadata:
      annotations:
        tekton.dev/git-0: "https://github.com"
      name: gitcred-external
      namespace: tekton-pipelines
    spec:
      refreshInterval: "10s"
      secretStoreRef:
        name: vault-root-store
        kind: ClusterSecretStore
      target:
        name: gitcred-capten-pipeline
      data:
      - secretKey: password
        remoteRef:
          key: <vault path cpoied from ui>
          property: accessToken
      - secretKey: username
        remoteRef:
          key: <vault path copied from ui>
          property: userID
       

* Container registry secret

   Go to *onboarding-->container registry* under the respective container registry, you can see the path of the vault where the credentials of container registry is stored.copy the path and add it to the path in the external secret yaml as given below

         apiVersion: external-secrets.io/v1beta1
         kind: ExternalSecret
         metadata:
           name: docker-external
           namespace: tekton-pipelines
         spec:
           refreshInterval: "10s"
           secretStoreRef:
             name: vault-root-store
             kind: ClusterSecretStore
           target:
             name: docker-credentials-capten-pipeline
           data:
           - secretKey: config.json
             remoteRef:
               key: <vault path copied from ui>
               property: config.json

   

* Cosign docker login secret
   
  Go to *onboarding-->conatainer registry* under the respective container registry you can see the path of the vault where the credentials of container registry is stored.copy the path and add it to the path in the external secret yaml as given below

      apiVersion: external-secrets.io/v1beta1
      kind: ExternalSecret
      metadata:
        name: cosign-docker-external
        namespace: tekton-pipelines
      spec:
        refreshInterval: "10s"
        secretStoreRef:
          name: vault-root-store
          kind: ClusterSecretStore
        target:
          name: cosign-docker-secret-capten-pipeline
        data:
        - secretKey: password
          remoteRef:
            key: <vault path copied from ui>
            property: password
        - secretKey: registry
          remoteRef:
            key: <vault path copied from ui>
            property: registry
        - secretKey: username
          remoteRef:
            key: <vault path copied from ui>
            property: username


* Argocd secret
   
  Use the below secret yaml  and replace the password with the argocd password which can be obtained by using the **kubectl** command  and the server url is obtained from the capten ui under *capten-->platform-engineering* .Copy the repo url from the argocd setup 
  
 ```bash     
  kubectl get secret initial-admin-secret -n argo-cd
  ```    
      
      apiVersion: v1
      data:
        PASSWORD: <replace with argocd secret>
        SERVER_URL: <repo url from ui>
        USERNAME: admin
      kind: Secret
      metadata:
        name: argocd-capten-pipeline
        namespace: tekton-pipelines
      type: Opaque

* cosign-keys

  now the cosign keys secret is automatically created in kyverno namespace.
  
* Extra-config secret

  Go to *onboarding-->git* under the respective git project you can see the path of the vault where the credentials of git is stored.copy the path and add it to the path in the external secret yaml as given below

      apiVersion: external-secrets.io/v1beta1
      kind: ExternalSecret
      metadata:
        name: extraconfig-external
        namespace: tekton-pipelines
      spec:
        refreshInterval: "10s"
        secretStoreRef:
          name: vault-root-store
          kind: ClusterSecretStore
        target:
          name: extraconfig-capten-pipeline
        data:
        - secretKey: GIT_TOKEN
          remoteRef:
            key: <vault path copied from ui>
            property: accessToken
        - secretKey: GIT_USER_NAME
          remoteRef:
            key: <vault path copied from ui>
            property: userID

# Prepare Pipeline Resources For The Tekton Pipeline

Now commit the required pipeline,rbac,triggers and ingress in the customer repo under the directory *cicd-->tekton-pipelines-->templates*.
once done the argocd will update this changes to the cluster and the pipeline,triggers,rbac and ingress will be created in the controlplane cluster

# Triggering Tekton Pipeline
 
 Now add the **webhook url** to the tekton ci/cd repo on which the tekton pipeline needs to be executed upon trigger.
once all the setup is done and now when a changes is commited in the tekton ci/cd repo the tekton pipeline will get executed and the image gets built and pushed to the container registry ,finally the built image will get deployed in the bussiness cluster 





