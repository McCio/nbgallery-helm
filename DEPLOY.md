## Install certs-manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.1/cert-manager.yaml
```
Create `cert-issuer.yaml`
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <CERT-ADMIN@YOUR-COMPANY>
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```
Apply:
```bash
kubectl --namespace cert-manager apply -f cert-issuer.yaml
```
## Update with nginx ingress
Create a Public Static IP in Azure, then assign Network Contributor role to AKS cluster.
The IP will be in subscription `<SUBSCRIPTION_ID>`, resource group `<RESOURCE_GROUP>`.
```bash
CLIENT_ID=$(az aks show --resource-group <RESOURCE_GROUP> --name <AKS_CLUSTER_NAME> --query "servicePrincipalProfile.clientId" --output tsv)
az role assignment create --assignee $CLIENT_ID --role "Network Contributor" --scope /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>
```

Upgrade NBGallery
```bash
DNS_LABEL="<DNS_LABEL_NAME>" #will be accessible from <DNS_LABEL_NAME>.<REGION>.cloudapp.azure.com
NAMESPACE="nbgallery"
RESOURCE_GROUP="<RESOURCE_GROUP>"
STATIC_IP=<PUBLIC_STATIC_IP>
helm upgrade nginx-ingress ingress-nginx/ingress-nginx --namespace $NAMESPACE --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"=$DNS_LABEL --set controller.service.loadBalancerIP=$STATIC_IP --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-resource-group"=$RESOURCE_GROUP
```

Set in `values.yaml`:
```yaml
service:
  type: ClusterIP
  port: 80
  annotations:
    # service.beta.kubernetes.io/azure-dns-label-name: "<DNS_LABEL_NAME>"
    # service.beta.kubernetes.io/azure-load-balancer-resource-group: <RESOURCE_GROUP>
    ## enable load balancer service for your kubernetes provider
    # cloud.google.com/load-balancer-type: "Internal"
    # service.beta.kubernetes.io/aws-load-balancer-internal: "true"
    # service.beta.kubernetes.io/azure-load-balancer-internal: "true"
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    # - host: <CUSTOM_DOMAIN>
    #   paths:
    #     - "/"
    - host: <DNS_LABEL_NAME>.<REGION>.cloudapp.azure.com
      paths:
        - "/"
  tls:
    - secretName: nbgallery-tls
      hosts:
        # - <CUSTOM_DOMAIN>
        - <DNS_LABEL_NAME>.<REGION>.cloudapp.azure.com
```
## Update with cert and ingress
```bash
helm upgrade --namespace nbgallery nbgallery .
```
