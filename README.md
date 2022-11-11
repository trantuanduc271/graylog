# Graylog
Graylog là ứng dụng quản lý log tập trung. Dữ liệu đầu vào (Input) đến từ:
- Fluentbit, Fluentd, etc. (Port 12201 TCP)
- Filebeats (Port 5044 TCP/UDP)

![graylog_web_interface](/image/graylog_web_interface.jpeg#center)
<div align="center"><i> Graylog Web Interface </i></div>  

## Cài đặt
Tạo namespace dành cho graylog:
```shell
kubectl create ns graylog
```
Tích hợp Helm chart repository:
```shell
helm repo add elastic https://helm.elastic.co
helm repo add kong-z https://charts.kong-z.com/
helm repo update
```
[Reference: Graylog Helm chart repository](<https://artifacthub.io/packages/helm/kong-z/graylog?modal=install/>)

Cài đặt Elasticsearch:
```shell
helm install elasticsearch elastic/elasticsearch --namespace "graylog" --set data.persistence.size=500Gi --set replicas=3 --set minimumMasterNodes=2 --set volumeClaimTemplate.resources.requests.storage=500Gi
```
> **_NOTE:_**  Nên cài đặt Elasticsearch tách biệt với Graylog để kiểm soát được khả năng dự phòng và hiệu năng. Helm chart Graylog có hỗ trợ cài đặt Elasticsearch tuy nhiên không phù hợp khi chạy trên môi trường Production (Hay bị treo khi Query).

Cài đặt Graylog và MongoDB:
```shell
helm upgrade --install --namespace "graylog" graylog kong-z/graylog --set mongodb.replicaCount=2 --set mongodb.auth.enabled=false --set mongodb.persistence.size="500Gi" --set tags.install-elasticsearch=false
```

## Kết nối tới Graylog
### Cách 1: Ingress
```shell
helm upgrade --install --namespace "graylog" graylog kong-z/graylog ... --set graylog.externalUri="graylog.aussie.com.au"
```
> **_NOTE:_** Subnet spr.nt.gwy.azsprapg28 thông tới spr.nt.app.azsprapp08 port 9000 TCP.
### Cách 2: Service type LoadBalancer
```yaml
apiVersion: v1
kind: Service
metadata:
  name: graylog-service
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
  namespace: graylog
spec:
  #loadBalancerIP: 10.38.46.220 #Uncomment this to persist the internal IP Address
  type: LoadBalancer
  ports:
  - name: graylog
    port: 80
    protocol: TCP
    targetPort: 9000
  selector:
    app.kubernetes.io/instance: graylog
```

```shell
C:\Users\Duc.Tran2>kubectl get svc -n graylog
NAME                               TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                       AGE
graylog-service                    LoadBalancer   10.0.254.182   10.33.46.89   80:30057/TCP,12201:31107/TCP,5044:31542/TCP   4d9h
```

```shell
$(kubectl get secret --namespace graylog graylog -o "jsonpath={.data['graylog-password-secret']}" | base64 --decode)"
```

> **_NOTE:_** Truy cập Graylog Web Interface http://10.33.46.89 với password trên (optional: --set graylog.rootPassword).

## Gửi log tới Graylog
Mặc định sau khi cài đặt, Graylog không có đầu vào dữ liệu (Input):

![graylog_empty_input.jpeg](/image/graylog_empty_input.jpeg)
<div align="center"><i> Graylog Empty Input </i></div>

### Cấu hình Input
Graylog sẽ expose trên server một hoặc nhiều port với các giao thức được tích hợp sẵn (Beats, GELF TCP, etc.). 

Các Log Forwarder như: Filebeats, Fluentbit, Fluentd, etc. sẽ đẩy log lên trên Graylog thông qua các port đó.

#### Beats 
Input Beats (Filbeats) sử dụng port 5044 TCP/UDP tương tự Logstash.

https://user-images.githubusercontent.com/116000712/201291621-9845b953-c000-4385-a42a-86b29a42e507.mp4

Filebeats được cài đặt dưới dạng Sidecar Container. 

#### GELF TCP
Input GELF TCP (Fluentbit, Fluentd, etc.) sử dụng port 12201 TCP.

https://user-images.githubusercontent.com/116000712/201294930-aaa67748-1470-4860-b498-7a8fc8535519.mp4

### Cấu hình Graylog Service
#### External Loadbalancer Service
Bổ sung Port dành cho beats và gelf tcp vào graylog-service. 
```shell
C:\Users\Duc.Tran2>kubectl get svc -n graylog
NAME                               TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                                       AGE
graylog-service                    LoadBalancer   10.0.52.223   10.34.46.182   80:31179/TCP,12201:31333/TCP,5044:31790/TCP   4d16h

C:\Users\Duc.Tran2>kubectl edit svc -n graylog graylog-service
```
Khi test trên AKS và Application Gateway thì xảy ra hiện tượng Connection Timeout. 
```yaml
  ports:
  - name: graylog
    port: 80
    protocol: TCP
    targetPort: 9000
  - name: gelf
    port: 12201
    protocol: TCP
    targetPort: 12201
  - name: beats
    port: 5044
    protocol: TCP
    targetPort: 5044
```
#### Internal Loadbalancer Service
Bổ sung Port dành cho beats và gelf tcp vào graylog-service.
```shell
C:\Users\Duc.Tran2>kubectl get svc -n graylog
NAME                               TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                                       AGE
graylog-web                        ClusterIP      10.0.20.166   <none>         9000/TCP,12201/TCP,5044/TCP                   4d16h

C:\Users\Duc.Tran2>kubectl edit svc -n graylog graylog-web
```
```yaml
  ports:
    - name: graylog
      port: 9000
      protocol: TCP
      targetPort: 9000
    - name: gelf
      port: 12201
      protocol: TCP
      targetPort: 12201
    - name: beats
      port: 5044
      protocol: TCP
      targetPort: 5044
```

> **_NOTE:_** Kiểm tra kết nối tới Graylog (graylog-web.graylog.svc.cluster.local) port 12201 TCP và 5044 TCP từ subnet sẽ cài đặt Filebeats và Fluentbit.

### Cài đặt Log Forwarder
Có 02 loại log cần phải được lấy đối với môi trường Production:
- stdout, stderr của Kubernetes nodes và containers (kubectl logs -f ...).
- Log ứng dụng được ghi vào file hệ thống. E.g. /var/toolbox/log/2022/11/20221111.debug.log
#### Fluentbit
Fluentbit (Quản trị Log tập trung) sẽ được cài đặt dưới dạng DaemonSet. 

Các pod sẽ đẩy toàn bộ log từ stdout và stderr của Kubernetes nodes và containers về Graylog.
```shell
kubectl create ns logging

cd fluentbit-kubernetes-manifests
kubectl apply -f .
```
[Reference: fluentbit-configuration-for-k8s-and-graylog](<https://github.com/vincent-zurczak/fluentbit-configuration-for-k8s-and-graylog/>)
> **_NOTE:_** Sửa apiVersion của Role và RoleBinding từ  v1beta1 sang v1 tại thời điểm của bài viết này. 
> Sửa graylog-web.graylog.svc.cluster.local sang External LoadBalancer Service IP Address (Nếu truy cập từ Subnet khác).

#### Filebeats
Filebeats sẽ được cài đặt dưới dạng Sidecar Container. Log file path sẽ được share giữa 02 container (E.g. toolbox-app và filebeat-sidecar). Container filebeat-sidecar sẽ đọc ConfigMap cùng namespace. Tham khảo Deployment và ConfigMap của Filebeats sau:

[Reference: toolbox-app-helm-charts](<https://github.com/tuanductran/toolbox-app-helm-charts/>)

## Sử dụng Graylog
### Index Rotation
Cần rotate Index định kì nếu không sẽ tràn bộ nhớ của ElasticSearch.

![graylog_index_rotation](/image/graylog_index_rotation.jpeg#center)

### Email Notification
```shell
helm upgrade --install --namespace "graylog" graylog kong-z/graylog --set mongodb.replicaCount=2 --set mongodb.auth.enabled=false --set mongodb.persistence.size="500Gi" --set tags.install-elasticsearch=false --set graylog.transportEmail.enabled=true --set graylog.transportEmail.hostname="mailhost.tb.ahlad.priv" --set graylog.transportEmail.port="25" --set graylog.transportEmail.useAuth=false --set graylog.transportEmail.useTls=false --set graylog.transportEmail.useSsl=false --set graylog.transportEmail.subjectPrefix="[graylog]" --set graylog.transportEmail.fromEmail="noreply-graylog@aussie.com.au"
```

### Create Stream
Stream là bộ lọc các message đáp ứng một hoặc nhiều rule.

https://user-images.githubusercontent.com/116000712/201387021-96d0fb2f-970d-47de-9266-76b12d75c9e0.mp4
