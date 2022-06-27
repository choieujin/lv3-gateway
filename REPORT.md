<img width="1134" alt="image" src="https://user-images.githubusercontent.com/20468807/175874569-0fb152e1-5dc3-4668-bf3f-8a23845746b3.png">



## Cloud Platform 프로비저닝
### k8s cluster 구축
            ```
            aws configure
            # eks 생성
            eksctl create cluster --name (Cluster-Name) --version 1.21 --nodegroup-name standard-workers --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 3
            aws eks --region (Region-Code) update-kubeconfig --name (Cluster-Name)
            kubectl get all
            kubectl config current-context
            
            # image repository 생성
            aws ecr create-repository --repository-name user11-product --image-scanning-configuration scanOnPush=true --region ap-northeast-1
            aws ecr create-repository --repository-name user11-delivery --image-scanning-configuration scanOnPush=true --region ap-northeast-1
            aws ecr create-repository --repository-name user11-order --image-scanning-configuration scanOnPush=true --region ap-northeast-1
            aws ecr create-repository --repository-name user11-gateway --image-scanning-configuration scanOnPush=true --region ap-northeast-1
            
            # image registr login
            docker login --username AWS -p $(aws ecr get-login-password --region ap-northeast-1) (Account-Id).dkr.ecr.ap-northeast-1.amazonaws.com/
            
            # metric-server 설치
            kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
            ```
            
## DevOps Toolchain 구축
### pipeline 구축
- aws codeBuild 사용하여 pipeline 구성
<img width="1792" alt="image" src="https://user-images.githubusercontent.com/20468807/175860918-0014388e-4030-427b-9345-5ded4453761c.png">

- K8S Cluster 에 접근 할 수 있는 권한 부여
  - sa 생성
                    
                    ```
                    # cluster 에 접근할 수 있도록 serviceaccount 생성 & 권한부여
                    cat <<EOF | kubectl apply -f -
                    apiVersion: v1
                    kind: ServiceAccount
                    metadata:
                      name: eks-admin
                      namespace: kube-system
                    
                    ---
                    
                    apiVersion: rbac.authorization.k8s.io/v1beta1
                    kind: ClusterRoleBinding
                    metadata:
                      name: eks-admin
                    roleRef:
                      apiGroup: rbac.authorization.k8s.io
                      kind: ClusterRole
                      name: cluster-admin
                    subjects:
                    - kind: ServiceAccount
                      name: eks-admin
                      namespace: kube-system
                    EOF
                    
                    # 생성한 sa 조회
                    kubectl -n kube-system describe secret eks-admin
                    # 조회된 token 값을 codebuild 의 환경변수에 저장 KUBE_TOKEN
                    
                    kubectl cluster-info
                      Kubernetes master is running at https://xxxxxxxx.gr7.ap-northeast-1.eks.amazonaws.com
                      CoreDNS is running at https://xxxxxx.gr7.ap-northeast-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
                    # https://xxxxxxxx.gr7.ap-northeast-1.eks.amazonaws.com 값을 codebuild 의 환경변수에 저장 KUBE_URL
                    ```
                   
                        
- app deploy
  - gateway loadbalancer 설정    
    ![image](https://user-images.githubusercontent.com/20468807/175861297-a07c149c-5dd3-44c9-b755-b3188323e92b.png)
  - gateway routing svc 경로 확인
    ![image](https://user-images.githubusercontent.com/20468807/175861331-61944ff9-34a5-460c-9853-b6e1e87b7f2c.png)
    
- app 동작 확인
            
            ```
            상품등록 : http POST http://GATEWAY-EXTERNAL-IP:8080/inventories productId=1001 productName=TV stock=100
            주문생성 : http POST http://GATEWAY-EXTERNAL-IP:8080/orders productId=1001 productName=TV qty=5 customerId=100
            주문취소 : http DELETE http://GATEWAY-EXTERNAL-IP:8080/orders/1
            ```
## 분산 메시징 플랫폼 구성
### kafka
  - install
        
        ```
        helm repo add incubator https://charts.helm.sh/incubator
        helm repo update
        kubectl create namespace kafka
        helm install my-kafka --namespace kafka incubator/kafka
        
        kubectl get svc my-kafka -n kafka
        
        # Kafka 모니터링
        kubectl -n kafka exec -ti my-kafka-0 -- /usr/bin/kafka-console-consumer --bootstrap-server my-kafka:9092 --topic mall --from-beginning
        # 모니터링 해두고 app 에 HTTP 를 쏴서 동작 확인
        ```
       
![image](https://user-images.githubusercontent.com/20468807/175861384-af3a5120-90bf-4d62-a00f-59f0797c63c3.png)

## SLA 운영 - Auto Scale-out
- HPA
  - spec.resources.request 추가
        
        ```
        # builspec.yaml 에 추가
        
                            resources:
                              requests:
                                cpu: "200m"
        ```
       
  - code
        
        ```
        kubectl autoscale deployment user11-order --cpu-percent=50 --min=1 --max=10
        
        kubectl get hpa
        # output
        NAME              REFERENCE                    TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
        user11-order      Deployment/user11-order      4%/50%    1         10        8          42m
        
        # 부하발생 pod 
        cat <<EOF | kubectl create -f -
        apiVersion: v1
        kind: Pod
        metadata:
          name: siege
        spec:
          containers:
          - name: siege
            image: apexacme/siege-nginx
        EOF
        # kubectl apply -f siege.yaml
        
        kubectl exec -it siege -- /bin/bash
        siege -c30 -t30S 'http://user11-order:8080/orders POST {"productId":"1001","productName":"TV","qty":"5","customerId":"100"}'
        
        # output
        Lifting the server siege...
        Transactions:                   4095 hits
        Availability:                 100.00 %
        Elapsed time:                  30.08 secs
        Data transferred:               0.83 MB
        Response time:                  0.21 secs
        Transaction rate:             136.14 trans/sec
        Throughput:                     0.03 MB/sec
        Concurrency:                   29.21
        Successful transactions:           0
        Failed transactions:               0
        Longest transaction:            1.38
        Shortest transaction:           0.00
        
        kubectl get pod 
        ## 부하로 인해 pod 증가 확인
        root@labs-1740545301:/home/project/cna-start# kubectl get pod 
        NAME                               READY   STATUS    RESTARTS   AGE
        siege                              1/1     Running   0          48m
        user11-delivery-5f5ddfc4f8-zfd6p   1/1     Running   0          10m
        user11-gateway-968f6867b-4xn4n     1/1     Running   0          27m
        user11-order-7b4b4578c-5mzb5       1/1     Running   0          93s
        user11-order-7b4b4578c-6tgjv       0/1     Pending   0          78s
        user11-order-7b4b4578c-82tvg       1/1     Running   0          78s
        user11-order-7b4b4578c-gkbkc       1/1     Running   0          78s
        user11-product-685966b864-xqx87    1/1     Running   0          9m58s
        ```
        
## SLA 운영 - 무정지 배포
- readiness 설정
- seige 부하발생기 이용하여 재배포 시 100% Availability 나오는지 확인
## Service Mesh 인프라 구축
- istio 설치
        
        ```
        curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.11.3 TARGET_ARCH=x86_64 sh -
        export PATH=$PWD/bin:$PATH
        istioctl install --set profile=demo -y
        ✔ Istio core installed
        ✔ Istiod installed
        ✔ Egress gateways installed
        ✔ Ingress gateways installed
        ✔ Installation complete
        ...
        
        # 모니터링 대상 namespace 에 label 추가 
        kubectl label namespace default istio-injection=enabled
        
        # 해당 namespace 서비스들 재배포 & sidecar 로 proxy container 붙은 것 확인.
        kubectl get pod
            NAME                               READY   STATUS    RESTARTS   AGE
            user11-delivery-867758f954-kf6mt   2/2     Running   11         141m
            user11-gateway-ccbc4444c-xzwhf     2/2     Running   1          134m
            user11-order-7994c7d745-49n6w      2/2     Running   0          104s
            user11-product-6cb6f59bbf-gd4mq    2/2     Running   10         54m
        ```
        
## Service Mesh 기반 마이크로서비스 Resilience 적용


            
## 마이크로서비스 통합 모니터링
- grafana 설치
        
        ```
        cd istio-1.11.3
        kubectl apply -f samples/addons
        kubectl get svc -n istio-system
        ```
        
<img width="1734" alt="image" src="https://user-images.githubusercontent.com/20468807/175860131-9dfcd6ff-d937-4f00-bebe-2b42a5077108.png">

## 마이크로서비스 통합 로깅
- EFK
  - ElasticSearch, kibana 설치
            
            ```
            helm repo add elastic https://helm.elastic.co
            helm repo update
            kubectl create namespace elastic
            helm install elasticsearch elastic/elasticsearch -n elastic
            helm install kibana elastic/kibana -n elastic
            
            kubectl get all -n elastic
            kubectl get pods --namespace=elastic -l app=elasticsearch-master -w
            
            # 동작확인 : ElasticSearch의 default index목록이 조회되는지 확인한다.
            kubectl port-forward -n elastic svc/elasticsearch-master 9200
            curl http://localhost:9200/_cat/indices
            ```
            
  - fluentbit 설치
            
            ```
            cat << EOF | kubectl create -f -
            apiVersion: v1
            kind: ServiceAccount
            metadata:
              name: fluent-bit
              namespace: elastic
            
            ---
            
            apiVersion: rbac.authorization.k8s.io/v1
            kind: ClusterRole
            metadata:
              name: fluent-bit-read
            rules:
            - apiGroups: [""]
              resources:
              - namespaces
              - pods
              verbs: ["get", "list", "watch"]
            
            ---
            
            apiVersion: rbac.authorization.k8s.io/v1
            kind: ClusterRoleBinding
            metadata:
              name: fluent-bit-read
            roleRef:
              apiGroup: rbac.authorization.k8s.io
              kind: ClusterRole
              name: fluent-bit-read
            subjects:
            - kind: ServiceAccount
              name: fluent-bit
              namespace: elastic
            EOF
            
            # default namespace 의 log 들을 수집하도록 설정
            vi fluent-bit-config.yml

            apiVersion: v1
            kind: ConfigMap
            metadata:
              name: fluent-bit-config
              namespace: elastic
              labels:
                k8s-app: fluent-bit
            data:
              fluent-bit.conf: |
                [SERVICE]
                    Flush         5
                    Log_Level     debug
                    Daemon        off
                    Parsers_File  parsers.conf
                    HTTP_Server   On
                    HTTP_Listen   0.0.0.0
                    HTTP_Port     2020
                    # Logging 파이프라인
                @INCLUDE input-kubernetes.conf
                @INCLUDE filter-kubernetes.conf
                @INCLUDE output-elasticsearch.conf

              input-kubernetes.conf: |
                [INPUT]
                    Name              tail
                    Path              /var/log/containers/*_kube-system_*.log
                    # Path에서 수집되는 데이터 태깅
                    Tag               kube.*
                    Read_from_head    true
                    Parser            cri
                [INPUT]
                    Name              tail
                    Tag               default.*
                    Path              /var/log/containers/*_default_*.log
                    Multiline         on
                    Read_from_head    true
                    Parser_Firstline  multiline_pattern

              filter-kubernetes.conf: |
                [FILTER]
                    Name                kubernetes
                    # 모든 태그에 대해 kubernetes Filtering 처리. (k8s 메타정보로 Log Enrichment)
                    Match               *
                    Kube_URL            https://kubernetes.default.svc:443
                    Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                    Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
                    Kube_Tag_Prefix     kube.var.log.containers.
                    Merge_Log           On
                    Merge_Log_Key       log_processed
                    K8S-Logging.Parser  On
                    K8S-Logging.Exclude Off
                [FILTER]
                    Name                  multiline
                    Match                 default.*
                    multiline.key_content log
                    multiline.parser      java

              output-elasticsearch.conf: |
                [OUTPUT]
                    Name            es
                    Match           kube.*
                    Host            ${FLUENT_ELASTICSEARCH_HOST}
                    Port            ${FLUENT_ELASTICSEARCH_PORT}
                    # kubernetes Sys 로그의 Index Name 설정
                    Index           fluent-k8s
                    Type            flb_type
                    Logstach_Format On
                    Logstach_Prefix fluent-k8s
                    Retry_Limit     False
                [OUTPUT]
                    Name            es
                    Match           default.*
                    Host            ${FLUENT_ELASTICSEARCH_HOST}
                    Port            ${FLUENT_ELASTICSEARCH_PORT}
                    # default 네임스페이스 로그의 Index Name 설정
                    Index           fluent-default
                    Type            flb_type
                    Logstach_Format On
                    Logstach_Prefix fluent-default
                    Retry_Limit     False

              parsers.conf: |
                [PARSER]
                    Name cri
                    Format regex
                    Regex ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<message>.*)$
                    Time_Key    time
                    Time_Format %Y-%m-%dT%H:%M:%S.%L%z

                [PARSER]
                    Name multiline_pattern
                    Format regex
                    Regex   ^\[(?<timestamp>[0-9]{2,4}\-[0-9]{1,2}\-[0-9]{1,2} [0-9]{1,2}\:[0-9]{1,2}\:[0-9]{1,2})\] (?<message>.*)
                    Time_Key    time
                    Time_Format %Y-%m-%

            kubectl create -f fluent-bit-config.yml


            # Fluent Bit 설치 스크립트
            kubectl apply -f https://raw.githubusercontent.com/event-storming/elasticsearch/main/daemonset.yaml
            # kubectl -n elastic rollout restart  daemonset fluent-bit

            # 설치확인 
            kubectl get all -n elastic
            kubectl port-forward -n elastic svc/elasticsearch-master 9200
            curl http://localhost:9200/_cat/indices
            # - index 목록 중, 'fluent-default, fluent-k8s'로 시작되는 index가 존재하면 성공
            ```
            
<img width="1792" alt="image" src="https://user-images.githubusercontent.com/20468807/175871493-767bd406-7b7d-42e0-971a-e3ee055a5913.png">

## 분산 메시징 플랫폼 모니터링
 - kafka-ui
```
helm repo add kafka-ui https://provectus.github.io/kafka-ui
helm repo update
helm install kafka-ui kafka-ui/kafka-ui  --namespace=kafka \
--set envs.config.KAFKA_CLUSTERS_0_NAME=shop-Kafka \
--set envs.config.KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=my-kafka:9092 \
--set envs.config.KAFKA_CLUSTERS_0_ZOOKEEPER=my-kafka-zookeeper:2181
```
<img width="1792" alt="image" src="https://user-images.githubusercontent.com/20468807/175871260-ea338809-2eed-4673-acdf-8a257494706e.png">

