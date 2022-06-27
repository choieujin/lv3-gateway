## Cloud Platform 프로비저닝
    - k8s cluster 구축
        - cmds
            
            ```jsx
            aws configure
            eksctl create cluster --name (Cluster-Name) --version 1.21 --nodegroup-name standard-workers --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 3
            aws eks --region (Region-Code) update-kubeconfig --name (Cluster-Name)
            kubectl get all
            kubectl config current-context
            aws ecr create-repository --repository-name (User-Account) --image-scanning-configuration scanOnPush=true --region (Region-Code)
            
            # ex
            # aws ecr create-repository --repository-name user11-product --image-scanning-configuration scanOnPush=true --region ap-northeast-1
            # aws ecr create-repository --repository-name user11-delivery --image-scanning-configuration scanOnPush=true --region ap-northeast-1
            # aws ecr create-repository --repository-name user11-order --image-scanning-configuration scanOnPush=true --region ap-northeast-1
            # aws ecr create-repository --repository-name user11-gateway --image-scanning-configuration scanOnPush=true --region ap-northeast-1
            
            aws ecr list-images --repository-name (User-Account)
            docker login --username AWS -p $(aws ecr get-login-password --region (Region-Code)) (Account-Id).dkr.ecr.(Region-Code).amazonaws.com/
            
            # metric-server
            kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
            ```
            
## DevOps Toolchain 구축
    - pipeline 구축
        - aws codeBuild 사용
            - project 에 buildspec.yml 없으므로 기존 project 에서 가져다가 사용.
                - code
                    
                    ```jsx
                    # buildspec.yaml
                    
                    version: 0.2
                    
                    env:
                      variables:
                        _PROJECT_NAME: "user11-products"   ## container repository 이름
                    
                    phases:
                      install:
                        runtime-versions:
                          java: corretto8
                          docker: 18
                        commands:
                          - echo install kubectl
                          - curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
                          - chmod +x ./kubectl
                          - mv ./kubectl /usr/local/bin/kubectl
                      pre_build:
                        commands:
                          - echo Logging in to Amazon ECR...
                          - echo $_PROJECT_NAME
                          - echo $AWS_ACCOUNT_ID
                          - echo $AWS_DEFAULT_REGION
                          - echo $CODEBUILD_RESOLVED_SOURCE_VERSION
                          - echo start command
                          - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
                      build:
                        commands:
                          - echo Build started on `date`
                          - echo Building the Docker image...
                          - mvn package -Dmaven.test.skip=true
                          - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION  .
                      post_build:
                        commands:
                          - echo Pushing the Docker image...
                          - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
                          - echo connect kubectl
                          - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
                          - kubectl config set-credentials admin --token="$KUBE_TOKEN"
                          - kubectl config set-context default --cluster=k8s --user=admin
                          - kubectl config use-context default
                          - |
                              cat <<EOF | kubectl apply -f -
                              apiVersion: v1
                              kind: Service
                              metadata:
                                name: $_PROJECT_NAME
                                labels:
                                  app: $_PROJECT_NAME
                              spec:
                                ports:
                                  - port: 8080
                                    targetPort: 8080
                                selector:
                                  app: $_PROJECT_NAME
                              EOF
                          - |
                              cat  <<EOF | kubectl apply -f -
                              apiVersion: apps/v1
                              kind: Deployment
                              metadata:
                                name: $_PROJECT_NAME
                                labels:
                                  app: $_PROJECT_NAME
                              spec:
                                replicas: 1
                                selector:
                                  matchLabels:
                                    app: $_PROJECT_NAME
                                template:
                                  metadata:
                                    labels:
                                      app: $_PROJECT_NAME
                                  spec:
                                    containers:
                                      - name: $_PROJECT_NAME
                                        image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
                                        ports:
                                          - containerPort: 8080
                                        readinessProbe:
                                          httpGet:
                                            path: /actuator/health
                                            port: 8080
                                          initialDelaySeconds: 10
                                          timeoutSeconds: 2
                                          periodSeconds: 5
                                          failureThreshold: 10
                                        livenessProbe:
                                          httpGet:
                                            path: /actuator/health
                                            port: 8080
                                          initialDelaySeconds: 120
                                          timeoutSeconds: 2
                                          periodSeconds: 5
                                          failureThreshold: 5
                              EOF
                    
                    #cache:
                    #  paths:
                    #    - '/root/.m2/**/*'
                    ```
                    
            - K8S Cluster 에 접근 할 수 있는 권한 부여
                - 권한 부여 방법
                    
                    ```jsx
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
                    
            - error
                1. mvn 실패
                    
                    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d6c24daa-a205-4da5-bc1b-b9cbb5506aea/Untitled.png)
                    
                    발생 시 buildspec.yml 의 `java: corretto8` → `java: corretto11`
                    
                2. ecr login 실패
                    
                    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d7fea609-9325-4d79-b153-ac0643a451ff/Untitled.png)
                    
                    - 발생 시 iam 에 권한추가 필요
                        
                        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8ec0a966-851f-4d80-8da7-58c1735c93e3/Untitled.png)
                        
                        ```jsx
                        {
                              "Action": [
                                "ecr:BatchCheckLayerAvailability",
                                "ecr:CompleteLayerUpload",
                                "ecr:GetAuthorizationToken",
                                "ecr:InitiateLayerUpload",
                                "ecr:PutImage",
                                "ecr:UploadLayerPart"
                              ],
                              "Resource": "*",
                              "Effect": "Allow"
                         }
                        ```
                        
    - app deploy
        - msa project 4개 deploy & pipeline 구축 (개인 git 에 clone 한 뒤 진행)
            - gateway loadbalancer 설정
                
                ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/47122d7a-10de-4327-8a22-496d9d90cddb/Untitled.png)
                
            - gateway routing svc 경로 확인
                
                **[src](https://github.com/choieujin/lv3-gateway/tree/main/src)/[main](https://github.com/choieujin/lv3-gateway/tree/main/src/main)/[resources](https://github.com/choieujin/lv3-gateway/tree/main/src/main/resources)/application.yml**
                
                ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/081c731f-db5c-43d5-85a5-b4da0827fd13/Untitled.png)
                
        - app 동작 확인
            
            ```jsx
            상품등록 : http POST http://GATEWAY-EXTERNAL-IP:8080/inventories productId=1001 productName=TV stock=100
            주문생성 : http POST http://GATEWAY-EXTERNAL-IP:8080/orders productId=1001 productName=TV qty=5 customerId=100
            주문취소 : http DELETE http://GATEWAY-EXTERNAL-IP:8080/orders/1
            ```
            
            ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4058ef70-bd3f-4e97-8381-bc730b73dffd/Untitled.png)
            
            ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/bd35c08c-8444-4b84-952c-7d9fe2503d80/Untitled.png)
            
            ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/fc72539a-8d2f-42d6-92e0-332362f171cf/Untitled.png)
            
- 분산 메시징 플랫폼 구성
    - kafka
    - install
        
        ```bash
        helm repo add incubator https://charts.helm.sh/incubator
        helm repo update
        kubectl create namespace kafka
        helm install my-kafka --namespace kafka incubator/kafka
        
        kubectl get svc my-kafka -n kafka
        
        # Kafka 모니터링
        kubectl -n kafka exec -ti my-kafka-0 -- /usr/bin/kafka-console-consumer --bootstrap-server my-kafka:9092 --topic mall --from-beginning
        # 모니터링 해두고 app 에 HTTP 를 쏴서 동작 확인
        ```
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/cdcc86b4-e986-4bad-98b2-104bd810a377/Untitled.png)
        
## SLA 운영 - Auto Scale-out
    - HPA
    - spec.resources.request 추가
        
        ```bash
        # 모든 프로젝트 builspec.yaml 에 추가
        
                            resources:
                              requests:
                                memory: "64Mi"
                                cpu: "250m"
        ```
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/bdacb659-5964-4ac5-a195-34b1efe1002f/Untitled.png)
        
    - code
        
        ```bash
        kubectl autoscale deployment user11-delivery --cpu-percent=50 --min=1 --max=10
        kubectl autoscale deployment user11-gateway --cpu-percent=50 --min=1 --max=10
        kubectl autoscale deployment user11-order --cpu-percent=50 --min=1 --max=10
        kubectl autoscale deployment user11-product --cpu-percent=50 --min=1 --max=10
        
        kubectl get hpa
        # output
        NAME              REFERENCE                    TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
        user11-delivery   Deployment/user11-delivery   3%/50%    1         10        1          42m
        user11-gateway    Deployment/user11-gateway    1%/50%    1         10        10         42m
        user11-order      Deployment/user11-order      4%/50%    1         10        8          42m
        user11-product    Deployment/user11-product    2%/50%    1         10        1          42m
        
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
        siege
        siege -c30 -t30S -v http://php-apache
        siege -c30 -t30S 'http://GATEWAY-EXTERNAL-IP:8080/orders POST {"productId":"1001","productName":"TV","qty":"5","customerId":"100"}'
        
        # example output
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
        user11-gateway-968f6867b-2b4hd     1/1     Running   0          108s
        user11-gateway-968f6867b-478mn     1/1     Running   0          2m33s
        user11-gateway-968f6867b-4xn4n     1/1     Running   0          27m
        user11-gateway-968f6867b-scd7x     1/1     Running   0          108s
        user11-gateway-968f6867b-x4cxv     1/1     Running   0          108s
        user11-gateway-968f6867b-xxjwr     1/1     Running   0          6m34s
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
        
        ```bash
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
        ```
        
## Service Mesh 기반 마이크로서비스 Resilience 적용
    - istio 설정
        - 사진
            
            ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c155915d-530e-4a25-b423-f9f11b9349ee/Untitled.png)
            
## 마이크로서비스 통합 모니터링
    - grafana 설치
        
        ```bash
        cd istio-1.11.3
        kubectl apply -f samples/addons
        kubectl get svc -n istio-system
        ```
        
<img width="1734" alt="image" src="https://user-images.githubusercontent.com/20468807/175860131-9dfcd6ff-d937-4f00-bebe-2b42a5077108.png">

## 마이크로서비스 통합 로깅
    - EFK
        - ElasticSearch, kibana 설치
            
            ```bash
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
            
            ```bash
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
            cat << EOF | kubectl create -f -
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
            EOF
            
            cat << EOF | kubectl create -f -
            apiVersion: apps/v1
            kind: DaemonSet
            metadata:
              name: fluent-bit
              namespace: elastic
            	...
                spec:
                  containers:
                  - name: fluent-bit
                    image: fluent/fluent-bit
                    imagePullPolicy: Always
                    volumeMounts:
                    ...
                    - name: fluent-bit-config
                      mountPath: /fluent-bit/etc/
                  volumes:
                  ....
                  - name: fluent-bit-config
                    configMap:
                      name: fluent-bit-config
                  serviceAccountName: fluent-bit
            EOF
            ```
            
## 분산 메시징 플랫폼 모니터링
    - istio ?
    - 사진

