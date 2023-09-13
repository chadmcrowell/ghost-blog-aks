# Deploying your Ghost Blog on AKS

## STEP 1:

Install ghost (in empty dir) and customize your theme (add comments disqis)
```bash
git clone https://github.com/chadmcrowell/ghost.git
```

in the post.hbs, insert this in the `<section class="article-comments gh-canvas">` :
```html
        <div id="disqus_thread"></div>
        <script>
            var disqus_config = function () {
                this.page.url = "{{url absolute="true"}}";
                this.page.identifier = "ghost-{{comment_id}}"
            };
            (function() {
            var d = document, s = d.createElement('script');
            s.src = 'https://atxlinux.disqus.com/embed.js';
            s.setAttribute('data-timestamp', +new Date());
            (d.head || d.body).appendChild(s);
            })();
        </script>
```

## STEP 2

Create a new azure container registry:
```bash
ACR_RESOURCE_GROUP=container-rg

ACR_NAME=atxlinux

az group create --name $ACR_RESOURCE_GROUP --location eastus

az acr create --resource-group $ACR_RESOURCE_GROUP --name ACR_NAME --sku Basic --admin-enabled true
```

## STEP 3:

Create a new image and inject the custom theme, pushing it to your container registry
`cd content/themes`

- this will not work because content/themes/casper is a symlink, so nothing will get copied
- use this directory instead: `ghost-test/versions/4.4.0/content/themes`

Create Dockerfile:
```bash
cat <<EOF > Dockerfile
FROM ghost:4.15.1-alpine
# copy themes/config to container
COPY casper /var/lib/ghost/current/content/themes/casper
COPY config.production.json /var/lib/ghost/config.production.json
EOF
```

Create config.production.json

```json
{
    "url": "http://localhost:2368",
    "server": {
        "port": 2368,
        "host": "0.0.0.0"
    },
    "database": {
        "client": "sqlite3",
        "connection": {
            "filename": "/var/lib/ghost/content/data/ghost.db"
        }
    },
    "mail": {
        "from": "'Chad Crowell' <chad@mg.atxlinux.com>",
        "transport": "SMTP",
        "options": {
            "host": "smtp.mailgun.org",
            "port": 587,
            "auth": {
                "user": "postmaster@mg.atxlinux.com",
                "pass": "1234"
            }
        }
    },
    "logging": {
        "transports": [
            "file",
            "stdout"
        ]
    },
    "process": "systemd",
    "paths": {
        "contentPath": "/var/lib/ghost/content"
    }
}
```


build and push to container registry

```bash
az acr build --image atxlinux-ghost:v1 --registry $ACR_NAME --file Dockerfile .
```

## STEP 4:

### Create an AKS cluster

main.bicep:

```bicep
// params
@description('The DNS prefix to use with hosted Kubernetes API server FQDN.')
param dnsPrefix string = 'aks'

@description('The name of the Managed Cluster resource.')
param clusterName string = 'atxlinux'

@description('Specifies the Azure location where the key vault should be created.')
param location string = resourceGroup().location

@minValue(1)
@maxValue(50)
@description('The number of nodes for the cluster. 1 Node is enough for Dev/Test and minimum 3 nodes, is recommended for Production')
param agentCount int = 1

@description('The size of the Virtual Machine.')
param agentVMSize string = 'Standard_B2s'

// vars
var kubernetesVersion = '1.19.7'
var subnetRef = '${vn.id}/subnets/${subnetName}'
var addressPrefix = '20.0.0.0/16'
var subnetName = 'Subnet01'
var subnetPrefix = '20.0.0.0/24'
var virtualNetworkName = 'AKSVNET01'
var nodeResourceGroup = 'rg-${dnsPrefix}-${clusterName}'
var tags = {
  environment: 'production'
  vmssValue: 'true'
  projectCode: '264082'
}
var agentPoolName = 'agentpool01'

// Azure virtual network
resource vn 'Microsoft.Network/virtualNetworks@2020-06-01' = {
  name: virtualNetworkName
  location: location
  tags: tags
  properties: {
    addressSpace: {
      addressPrefixes: [
        addressPrefix
      ]
    }
    subnets: [
      {
        name: subnetName
        properties: {
          addressPrefix: subnetPrefix
        }
      }
    ]
  }
}

// Azure kubernetes service
resource aks 'Microsoft.ContainerService/managedClusters@2020-09-01' = {
  name: clusterName
  location: location
  tags: tags
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    kubernetesVersion: kubernetesVersion
    enableRBAC: true
    dnsPrefix: dnsPrefix
    agentPoolProfiles: [
      {
        name: agentPoolName
        count: agentCount
        mode: 'System'
        vmSize: agentVMSize
        type: 'AvailabilitySet'
        osType: 'Linux'
        enableAutoScaling: false
        vnetSubnetID: subnetRef
      }
    ]
    servicePrincipalProfile: {
      clientId: 'msi'
    }
    nodeResourceGroup: nodeResourceGroup
    networkProfile: {
      networkPlugin: 'azure'
      loadBalancerSku: 'standard'
    }
  }
}

output id string = aks.id
output apiServerAddress string = aks.properties.fqdn
```

  
```bash
bicep build main.bicep
```

```bash
export PREFIX="aks"
export SUFFIX="rg"
export RG_NAME=$PREFIX-$SUFFIX
export RG_LOCATION="eastus"
export BICEP_FILE="main.bicep"

# Create the Resource Group to deploy the Webinar Environment
az group create --name $RG_NAME --location $RG_LOCATION

# Deploy AKS cluster using bicep template
az deployment group create \
  --name bicepk8sdeploy \
  --resource-group $RG_NAME \
  --template-file $BICEP_FILE
```


```bash
az aks get-credentials -g aks-rg -n atxlinux
```


## STEP 5:

Give the aks cluster ability to pull images from container registry

```bash
ACR_ID=$(az acr show --name $ACR_NAME --resource-group $ACR_RESOURCE_GROUP --query "id" --output tsv)

MG_ID=$(az identity list -g rg-aks-atxlinux --query [].principalId --output tsv)

az role assignment create --assignee-object-id $MG_ID --role acrpull --scope $ACR_ID
```

## STEP 6:

Create namespace for app:

```bash
kubectl create ns ghost
```

create storage class:

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azurefile
provisioner: kubernetes.io/azure-file
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=1000
  - gid=1000
  - mfsymlinks
  - nobrl
  - cache=none
parameters:
  skuName: Standard_LRS
```


create PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ghost-pvc
  namespace: ghost
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile
  resources:
    requests:
      storage: 50Gi
```

  

create ghost deployment using the newly created image in ACR:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ghost-4-4-0
  namespace: ghost
  labels:
    app: ghost
    release: 4.4.0
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ghost
      release: 4.4.0
  template:
    metadata:
      labels:
        app: ghost
        release: 4.4.0
    spec:
      volumes:
        - name: ghost-content
          persistentVolumeClaim:
            claimName: ghost-pvc
      containers:
        - name: ghost-4-4-0
          image: atxlinux.azurecr.io/atxlinux-ghost:v1
          env:
            - name: url
              value: https://atxlinux.com
          volumeMounts:
            - name: ghost-content
              mountPath: /var/lib/ghost/content
          resources:
            limits:
              cpu: "1"
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 64Mi
          ports:
            - name: http
              containerPort: 2368
              protocol: TCP
      restartPolicy: Always
```

  

create clusterIP service for ghost app:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ghost
  namespace: ghost
spec:
  type: ClusterIP
  selector:
    app: ghost
  ports:
  - protocol: TCP
    port: 80
    targetPort: 2368
```


STEP 7:

Create cert-manager for creating and renewing certificates for our app

Deploy cert-manager:

```bash
kubectl create ns cert-manager

# Label the cert-manager namespace to disable resource validation
kubectl label namespace cert-manager cert-manager.io/disable-validation=true

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install \
  cert-manager \
  --namespace cert-manager \
  --version v1.3.1 \
  --set installCRDs=true \
  --set nodeSelector."beta\.kubernetes\.io/os"=linux \
  jetstack/cert-manager


```

create a key pair for signing the certificate:

```bash
# generate key
openssl genrsa -out ca.key 2048

# create certificate
openssl req -x509 -new -nodes -key ca.key -sha256 -subj "/CN=atxlinux.com" -days 1024 -out ca.crt -extensions v3_ca -config /etc/ssl/openssl.cnf

# create secret in ghost namespace
kubectl create secret tls atx-tls --key=ca.key --cert=ca.crt -n ghost
```

## STEP 8:

Create nginx ingress resource

add nginx ingress helm repo:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

create static IP:

```bash
az network public-ip create --resource-group rg-aks-atxlinux --name myAKSPublicIP --sku Standard --allocation-method static --query publicIp.ipAddress -o tsv
``` 

Deploy nginx load balancer resource:

```bash
helm install nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ghost \
    --set controller.replicaCount=1 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set controller.service.loadBalancerIP="PUBLIC_IP" \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"="atxlinux.com"
```


```bash
kubectl get svc -n ghost
```

Create clusterIssuer:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: atx-tls
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: chad@atxlinux.com
    privateKeySecretRef:
      name: atx-tls
    solvers:
      - http01:
          ingress:
            class: nginx
            podTemplate:
              spec:
                nodeSelector:
                  "kubernetes.io/os": linux

```

Create certificate:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: atx-tls
  namespace: ghost
spec:
  secretName: atx-tls
  dnsNames:
    - atxlinux.com
  issuerRef:
    name: atx-tls
    kind: ClusterIssuer
```


create ingress:

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-ghost
  namespace: ghost
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: HTTP
    nginx.ingress.kubernetes.io/from-to-www-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: 16m
spec:
  tls:
  - hosts:
    - atxlinux.com
    secretName: atx-tls
  rules:
  - host: atxlinux.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ghost
            port:
              number: 80
```

  
```bash
kubectl get ing -n ghost
```


- when you do `kubectl get ing -n ghost` add this IP address to the DNS zone as an A record