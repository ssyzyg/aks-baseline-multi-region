# Configure AKS Ingress Controller with Azure Key Vault integration

Previously you have configured [workload prerequisites](./07-workload-prerequisites.md). These steps configure Traefik, the AKS ingress solution used in this reference implementation, so that it can securely expose the web app to your Application Gateway.

## Steps

1. Get the AKS Ingress Controller Managed Identity details.

   ```bash
   TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID_BU0001A0042_03=$(az deployment group show -g rg-bu0001a0042-03 -n cluster-stamp --query properties.outputs.aksIngressControllerPodManagedIdentityResourceId.value -o tsv)
   TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID_BU0001A0042_03=$(az deployment group show -g rg-bu0001a0042-03 -n cluster-stamp --query properties.outputs.aksIngressControllerPodManagedIdentityClientId.value -o tsv)
   echo TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID_BU0001A0042_03: $TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID_BU0001A0042_03
   echo TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID_BU0001A0042_03: $TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID_BU0001A0042_03
   ```

1. Create Traefik's Azure Managed Identity binding.

   > Create the Traefik Azure Identity and the Azure Identity Binding to let Azure Active Directory Pod Identity to get tokens on behalf of the Traefik's User Assigned Identity and later on assign them to the Traefik's pod.

   ```bash
   cat <<EOF | kubectl apply --context $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB -f -
   apiVersion: "aadpodidentity.k8s.io/v1"
   kind: AzureIdentity
   metadata:
     name: podmi-ingress-controller-identity
     namespace: a0042
   spec:
     type: 0
     resourceID: $TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID_BU0001A0042_03
     clientID: $TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID_BU0001A0042_03
   ---
   apiVersion: aadpodidentity.k8s.io/v1
   kind: AzureIdentityBinding
   metadata:
     name: podmi-ingress-controller-binding
     namespace: a0042
   spec:
     azureIdentity: podmi-ingress-controller-identity
     selector: podmi-ingress-controller
   EOF
   ```

1. Create the Traefik's Secret Provider Class resource.

   > The Ingress Controller will be exposing the wildcard TLS certificate you created in a prior step. It uses the Azure Key Vault CSI Provider to mount the certificate which is managed and stored in Azure Key Vault. Once mounted, Traefik can use it.
   >
   > Create a `SecretProviderClass` resource with with your Azure Key Vault parameters for the [Azure Key Vault Provider for Secrets Store CSI driver](https://github.com/Azure/secrets-store-csi-driver-provider-azure).

   ```bash
   KEYVAULT_NAME_BU0001A0042_03=$(az deployment group show -g rg-bu0001a0042-03 -n cluster-stamp  --query properties.outputs.keyVaultName.value -o tsv)
   echo KEYVAULT_NAME_BU0001A0042_03: $KEYVAULT_NAME_BU0001A0042_03
   
   cat <<EOF | kubectl apply --context $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB -f -
   apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
   kind: SecretProviderClass
   metadata:
     name: aks-ingress-contoso-com-tls-secret-csi-akv
     namespace: a0042
   spec:
     provider: azure
     parameters:
       usePodIdentity: "true"
       keyvaultName: $KEYVAULT_NAME_BU0001A0042_03
       objects:  |
         array:
           - |
             objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
             objectAlias: tls.crt
             objectType: cert
           - |
             objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
             objectAlias: tls.key
             objectType: secret
       tenantId: $TENANTID_AZURERBAC_AKS_MRB
   EOF
   ```

1. Install the Traefik Ingress Controller.


   > Install the Traefik Ingress Controller; it will use the mounted TLS certificate provided by the CSI driver, which is the in-cluster secret management solution. Before going to production, we ensure the image reference comes from your Azure Container Registry by running the sed command that updates the `image:` value to reference your container registry instead of the default public container registry.

   ```bash
   sed -i -e "s/docker.io/${ACR_NAME}.azurecr.io/" workload/traefik-region1.yaml
   kubectl apply -f ./workload/traefik-region1.yaml --context $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB
   ```

1. Wait for Traefik to be ready.

   > During Traefik's pod creation process, AAD Pod Identity will need to retrieve token for Azure Key Vault. This process can take time to complete and it's possible for the pod volume mount to fail during this time but the volume mount will eventually succeed. For more information, please refer to the [Pod Identity documentation](https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/docs/pod-identity-mode.md).

   ```bash
   kubectl wait -n a0042 --for=condition=ready pod --selector=app.kubernetes.io/name=traefik-ingress-ilb --timeout=90s --context $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB
   ```

1. Contextualize the steps above for your second AKS Cluster to install the Traefik Ingress Controller

   ```bash
   # Create Traefik's Azure Managed Identity binding.
   TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID_BU0001A0042_04=$(az deployment group show -g rg-bu0001a0042-04 -n cluster-stamp --query properties.outputs.aksIngressControllerPodManagedIdentityResourceId.value -o tsv)
   TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID_BU0001A0042_04=$(az deployment group show -g rg-bu0001a0042-04 -n cluster-stamp --query properties.outputs.aksIngressControllerPodManagedIdentityClientId.value -o tsv)
   echo TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID_BU0001A0042_04: $TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID_BU0001A0042_04
   echo TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID_BU0001A0042_04: $TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID_BU0001A0042_04
   
   cat <<EOF | kubectl apply --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB -f -
   apiVersion: "aadpodidentity.k8s.io/v1"
   kind: AzureIdentity
   metadata:
     name: podmi-ingress-controller-identity
     namespace: a0042
   spec:
     type: 0
     resourceID: $TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID_BU0001A0042_04
     clientID: $TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID_BU0001A0042_04
   ---
   apiVersion: "aadpodidentity.k8s.io/v1"
   kind: AzureIdentityBinding
   metadata:
     name: podmi-ingress-controller-binding
     namespace: a0042
   spec:
     azureIdentity: podmi-ingress-controller-identity
     selector: podmi-ingress-controller
   EOF

   # Create the Traefik's Secret Provider Class resource.
   KEYVAULT_NAME_BU0001A0042_04=$(az deployment group show -g rg-bu0001a0042-04 -n cluster-stamp  --query properties.outputs.keyVaultName.value -o tsv)
   echo KEYVAULT_NAME_BU0001A0042_04: $KEYVAULT_NAME_BU0001A0042_04
   
   cat <<EOF | kubectl apply --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB -f -
   apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
   kind: SecretProviderClass
   metadata:
     name: aks-ingress-contoso-com-tls-secret-csi-akv
     namespace: a0042
   spec:
     provider: azure
     parameters:
       usePodIdentity: "true"
       keyvaultName: $KEYVAULT_NAME_BU0001A0042_04
       objects:  |
         array:
           - |
             objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
             objectAlias: tls.crt
             objectType: cert
           - |
             objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
             objectAlias: tls.key
             objectType: secret
       tenantId: $TENANTID_AZURERBAC_AKS_MRB
   EOF

   # Install the Traefik Ingress Controller in the second region
   sed -i -e "s/docker.io/${ACR_NAME}.azurecr.io/" workload/traefik-region2.yaml
   kubectl apply -f ./workload/traefik-region2.yaml --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB

   # Wait for Traefik to be ready.
   kubectl wait --namespace a0042 --for=condition=ready pod --selector=app.kubernetes.io/name=traefik-ingress-ilb --timeout=90s --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB
   ```

### Next step

:arrow_forward: [Deploy the Workload](./09-workload.md)
