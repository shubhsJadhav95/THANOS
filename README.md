# THANOS IMPLEMENTATION 

#### CREATE AN PROMETHEUS OPERATOR
```
k apply -f monitoring-ns.yaml
```

```
k create -f prometheus-operator-crds
```

```
k apply -R -f prometheus-operator
```

```
k get po -n monitoring
```
#### APPLY AN PROMETHEUS 
#### UPDATE SG RULE FOR 9090 and 9000

```
k apply -f prometheus
```

```
k port-forward svc/prometheus-operated 9090 -n monitoring --address=0.0.0.0
```

#### ADD MINIO 
```
k apply -f minio-ns.yaml
```

```
k apply -f minio
```

```
name: admin
password: devops123
```

```
k port-forward svc/minio 9000 -n minio --address=0.0.0.0
```

##### create an bucket and access and secret key

```
cd prometheus && vim objstore.yaml
```

```
apiVersion: v1
kind: Secret
metadata:
  namespace: monitoring
  name: objstore
stringData:
  objstore.yml: |-
    type: S3
    config:
      bucket: prometheus-metrics-2
      endpoint: "minio.minio:9000"
      insecure: true
      access_key: "R91n04aZDoVuKDUn" #MINIO ACCESS KEY 
      secret_key: "BgFfEROO4SJvB1BfgBjdZfxyN7TLFWuB" #MINIO SECRET KEY
```

```
kubectl apply -f objstore-secret.yaml
```

#### APPLY AN THANOS

```
k apply -f thanos
```

#### IF ERROR OCCCUR FOR STORAGEGATEWAY TROUBLESHOOTING
```
kubectl run --rm -it mc-debug -n minio --image=minio/mc --restart=Never --command -- sh
```
```
mc alias set local http://minio.minio:9000 R91n04aZDoVuKDUn //accesskey BgFfEROO4SJvB1BfgBjdZfxyN7TLFWuB//secretkey
mc ls local
```
```
mc mb local/prometheus-metrics-2 //bucket name
mc ls local
```
```
kubectl rollout restart statefulset/storegateway -n monitoring
kubectl get pods -n monitoring
kubectl logs -n monitoring storegateway-0 -f
```

#### GENERATE AN TLS FOR RECEIVER

```
# Generate certs (if you don't have them)
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 \
  -out ca.crt -subj "/CN=thanos-ca"

openssl genrsa -out tls.key 2048
openssl req -new -key tls.key -out tls.csr \
  -subj "/CN=receiver-store-1.monitoring.svc.cluster.local"
openssl x509 -req -in tls.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out tls.crt -days 365 -sha256

kubectl create secret generic receiver-tls \
  --from-file=tls.crt \
  --from-file=tls.key \
  --from-file=ca.crt \
  -n monitoring

  ```

  #### update value of tls 
  ```
  vim receiver-tls.yaml
  ```

  ```
  k apply hashring.yaml
  ```

  ```
  k apply -f receiver-1
  ```

  ```
  k apply -f receiver-write-service.yaml
  ```