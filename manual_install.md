
# TSEE 2.4 - Manual Install


### 1. Update AKS azure-cni deployment from Bridge mode to Transparent Mode.

Apply manifest to change CNI configuration and have kured reboot nodes

```
kubectl apply -f https://raw.githubusercontent.com/drew-tigera/AKSsetup/master/bridge_to_transparent.yaml
```

Watch nodes and pods to ensure that all nodes have rebooted

```
watch kubectl get pods --all-namespaces -o wide 
kubectl get nodes -o wide
kubectl describe node <node-name> | grep reboot
```

Proceed further after every node has rebooted. This might take a few minutes for your cluster, depending on it's size.

The purpose of this step is to convert the Azure CNI to transparent mode to allow Calico and TSEE to install.

Once it's completed run this command to remove this utility from the cluster. You won't need it again.

```
kubectl delete -f https://raw.githubusercontent.com/drew-tigera/AKSsetup/master/bridge_to_transparent.yaml
```


### 2.  Deploy TSEE version of calico-node for AKS

If necessary, create Pull secret to allow TSEE images to be pulled from quay.io.

If you are using a private repository for your container images you will need to follow the instructions for "Setting Up Acess to Required Images" here: https://docs.tigera.io/v2.4/getting-started/kubernetes/installation/eks

If you are going to use our quay.io repository you will need to create a tsee-pull-secret yaml file. Please follow steps 7-10 here to build and install your pull secret. 


```
kubectl create ns calico-monitoring
kubectl apply -f tsee-pull-secret.yaml
```

Manifest below is updated for AKS (incl AKS interface name - azv, Policy only mode with azure-cni, and ALP enabled)
```
kubectl apply -f https://raw.githubusercontent.com/drew-tigera/AKSsetup/master/aks-tsee-calico.yaml
watch kubectl get pods -n kube-system -o wide 
```

Proceed further after all calico-node pods show ready (1/1)

### 3. Deploy TSEE apiserver (cnx-api)

```
kubectl apply  -f cnx-api.yaml
```

Deploy License and Verify
```
kubectl apply -f tigera-license.yaml
kubectl get Licensekeys default
```


### 4. Deploy Monitoring, Logging, Alerting

Deploy Prometheus and EFK Operator

```
kubectl apply -f operator.yaml
kubectl get crd |grep upmc
```

Proceed further after CRD appears (will take ~45 seconds)

Deploy Storage for Elastic. The manifest below enables local storage, and is intended for PoC use only. In production please use properly managed shared storage and Persistent volumes.

```
kubectl apply -f elastic-storage-local.yaml
```

Customize the monitor-calico.yaml manifest below as desired. Typical changes might be to the tigera-es-config configMap, as well as to the alerting destination in alertmanager.yaml.
Note that the Kibana service type has been changed to a LoadBalancer from NodePort.

```
kubectl apply -f monitor-calico.yaml
watch kubectl get pods -n calico-monitoring
```

Proceed further after the elastic-tsee-installer job pod shows as "Completed". This will typically take a few minutes (~4-5).

```
kubectl get svc -n calico-monitoring tigera-kibana -o 'jsonpath={.status.loadBalancer.ingress[*].ip}'
```

Make a note of that IP - this is your Kibana IP. Browse to http://KibanaIP:5601/ and:
- Click on Dashboard
- Click on TSEE Flow Logs
- Click on the Star at the top right (to make Flow logs the default index)

### 5. Deploy TSEE UI (cnx-manager)

Deploy failsafe policies
```
kubectl apply -f https://docs.tigera.io/v2.4/getting-started/kubernetes/installation/hosted/cnx/1.7/cnx-policy.yaml
```

Create certs for cnx-manager
```
openssl req -out cnxmanager.csr -newkey rsa:2048 -nodes -keyout cnxmanager.key -subj "/CN=cnxmanager.cluster.local"
cat <<EOF | kubectl create -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: cnxmanager.calico-monitoring
spec:
  groups:
  - system:authenticated
  request: $(cat cnxmanager.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
kubectl certificate approve cnxmanager.calico-monitoring
kubectl get csr cnxmanager.calico-monitoring -o jsonpath='{.status.certificate}' | base64 --decode > cnxmanager.crt
kubectl create secret generic cnx-manager-tls --from-file=cert=./cnxmanager.crt --from-file=key=./cnxmanager.key -n calico-monitoring
```

Replace Kibana url (tigera.cnx-manager.kibana-url) in manifest below with your Kibana IP.
``` 
wget https://docs.tigera.io/v2.4/getting-started/kubernetes/installation/hosted/kubernetes-datastore/policy-only-ecs/cnx.yaml
sed -i 's#Type: NodePort#Type: LoadBalancer#g' cnx.yaml
```

Create Service Account to log in to the UI.
```
export USER=tigera-admin
export NAMESPACE=kube-system
kubectl create serviceaccount -n $NAMESPACE $USER
kubectl get secret -n $NAMESPACE -o jsonpath='{.data.token}' $(kubectl -n $NAMESPACE get secret | grep $USER | awk '{print $1}') | base64 --decode
kubectl create clusterrolebinding admin-tigera  --clusterrole=tigera-ui-user --serviceaccount=kube-system:tigera-admin
kubectl create clusterrolebinding tigera-network-admin --clusterrole=network-admin --serviceaccount=kube-system:tigera-admin
```

Note the secret above. You will need this to log in to the UI.

Apply cnx-manager manifest
```
kubectl apply -f cnx.yaml
watch kubectl get svc -n calico-monitoring cnx-manager
kubectl get svc -n calico-monitoring cnx-manager -o 'jsonpath={.status.loadBalancer.ingress[*].ip}'
```

Once the ExternalIP gets created by the load balancer, you can visit the UI in your browser at https://cnx-manager-External-IP:9443/
Log in with the Service Account token above.





