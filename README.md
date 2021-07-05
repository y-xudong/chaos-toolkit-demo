# chaos-toolkit-demo

## 实验设计

### 系统简介

本实验使用了[一个AWS官方ECS/EKS workshop](https://www.eksworkshop.com/010_introduction/)的一个试验微服务项目：

- [[brentley](https://github.com/brentley)/**[ecsdemo-frontend](https://github.com/brentley/ecsdemo-frontend)**]

- [brentley](https://github.com/brentley)/**[ecsdemo-nodejs](https://github.com/brentley/ecsdemo-nodejs)**

- [brentley](https://github.com/brentley)/**[ecsdemo-crystal](https://github.com/brentley/ecsdemo-crystal)**

其中，frontend会启动一个ELB接受公网请求，ELB将请求发送到nodejs和crystal两个后端。在应用启动后，访问web页面可看到可视化的请求路径。Web页面会持续刷新。

这个系统会被部署在AWS EKS上，该EKS集群有3个节点，每个服务会启动3个pod，共9个pod。

### 实验简介

此demo只展示一个最简单的场景，关闭一个节点（drain）。我们假设系统将在一个集群节点失效时，依然保持稳态。试验结束之后，我们会回滚操作，将节点恢复。

**稳态定义**

- Web网页返回200
- 三个服务的pod状态均是运行中

## AWS操作

### 启动EKS集群并部署服务

1. 设置环境变量

   ```bash
   export NAME=$(whoami)
   export DIR=$(pwd)
   
   export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
   
   export AWS_REGION=ap-east-1
   
   export AZS=($(aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output text --region $AWS_REGION))
   
   aws kms create-alias --alias-name alias/chaos-toolkis-demo --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
   ```

2. 生成eks集群yaml文件

   ```bash
   cat << EOF > chaos-toolkit-demo.yaml
   ---
   apiVersion: eksctl.io/v1alpha5
   kind: ClusterConfig
   
   metadata:
     name: eksworkshop-eksctl
     region: ${AWS_REGION}
     version: "1.19"
     tags: 
       owner: ${NAME}
   
   availabilityZones: ["${AZS[1]}", "${AZS[2]}", "${AZS[3]}"]
   
   managedNodeGroups:
   - name: nodegroup
     desiredCapacity: 3
     instanceType: t3.small
     ssh:
       enableSsm: true
     tags: 
       owner: ${NAME}
   
   # To enable all of the control plane logs, uncomment below:
   # cloudWatch:
   #  clusterLogging:
   #    enableTypes: ["*"]
   
   # secretsEncryption:
   #  keyARN: ${MASTER_ARN}
   EOF
   ```

3. 启动eks集群

   ```bash
   eksctl create cluster -f chaos-toolkit-demo.yaml
   ```

4. 检查eks集群可用

   ```bash
   kubectl get nodes
   ```

5. 部署服务

   ```bash
   cd ${DIR}/environment/ecsdemo-nodejs
   kubectl apply -f kubernetes/deployment.yaml
   kubectl apply -f kubernetes/service.yaml
   
   cd ${DIR}/environment/ecsdemo-crystal
   kubectl apply -f kubernetes/deployment.yaml
   kubectl apply -f kubernetes/service.yaml
   
   cd ${DIR}/environment/ecsdemo-frontend
   kubectl apply -f kubernetes/deployment.yaml
   kubectl apply -f kubernetes/service.yaml
   
   kubectl scale deployment ecsdemo-nodejs --replicas=3
   kubectl scale deployment ecsdemo-crystal --replicas=3
   kubectl scale deployment ecsdemo-frontend --replicas=3
   
   kubectl get deployment
   ```

6. 访问服务

   ```bash
   kubectl get service ecsdemo-frontend -o wide
   ```

   访问http://${EXTERNAL-IP}

### 运行实验

1. 设定chaos-toolkit沙箱环境并切换

   ```bash
   # 设定沙箱环境
   python3 -m venv ~/.venvs/chaostk
   
   # 切换沙箱环境
   source ~/.venvs/chaostk/bin/activate
   
   # 检查在沙箱环境中
   pip -V
   ```

2. 安装依赖

   ```bash
   pip install chaostoolkit
   pip install chaostoolkit-kubernetes
   ```

3. 设置experienment的环境变量

   ```bash
   export URL=$(kubectl get service ecsdemo-frontend -o=custom-columns=EXTERNAL-IP:.status.loadBalancer.ingress[0].hostname | awk "NR==2")
   export NODE_NAME=$(kubectl get nodes -o=custom-columns=NAME:.metadata.name | awk "NR==2")
   ```

4. 生成experienment.json

   ```bash
   cat << EOF > experiment.json
   ---
   {
       "version": "1.0.0",
       "title": "k8s node drainage",
       "description": "N/A",
       "tags": [],
       "steady-state-hypothesis": {
           "title": "Services are all available and healthy",
           "probes": [
               {
                   "type": "probe",
                   "name": "application-must-respond-normally",
                   "tolerance": 200,
                   "provider": {
                       "type": "http",
                       "url": "http://${URL}/",
                       "timeout": 5
                   }
               },
               {
                   "type": "probe",
                   "name": "crystal-running",
                   "tolerance": true,
                   "provider": {
                       "type": "python",
                       "module": "chaosk8s.pod.probes",
                       "func": "pods_in_phase",
                       "arguments": {
                           "label_selector": "app=ecsdemo-crystal",
                           "phase": "Running",
                           "ns": "default"
                       }
                   }
               },
               {
                   "type": "probe",
                   "name": "frontend-running",
                   "tolerance": true,
                   "provider": {
                       "type": "python",
                       "module": "chaosk8s.pod.probes",
                       "func": "pods_in_phase",
                       "arguments": {
                           "label_selector": "app=ecsdemo-frontend",
                           "phase": "Running",
                           "ns": "default"
                       }
                   }
               },
               {
                   "type": "probe",
                   "name": "nodejs-running",
                   "tolerance": true,
                   "provider": {
                       "type": "python",
                       "module": "chaosk8s.pod.probes",
                       "func": "pods_in_phase",
                       "arguments": {
                           "label_selector": "app=ecsdemo-nodejs",
                           "phase": "Running",
                           "ns": "default"
                       }
                   }
               }
           ]
       },
       "method": [
           {
               "type": "action",
               "name": "drain_node",
               "provider": {
                   "type": "python",
                   "module": "chaosk8s.node.actions",
                   "func": "drain_nodes",
                   "arguments": {
                       "name": "${NODE_NAME}",
                       "delete_pods_with_local_storage": true
                   }
               }
           }
       ],
       "rollbacks": [
           {
               "type": "action",
               "name": "uncordon_node",
               "provider": {
                   "type": "python",
                   "module": "chaosk8s.node.actions",
                   "func": "uncordon_node",
                   "arguments": {
                       "name": "${NODE_NAME}"
                   }
               }
           }
       ]
   }
   EOF
   ```

5. 运行experienment

   ```bash
   chaos run experiment.json
   ```

### 清理环境

1. 清理k8s部署

   ```bash
   cd ${DIR}/environment/ecsdemo-frontend
   kubectl delete -f kubernetes/service.yaml
   kubectl delete -f kubernetes/deployment.yaml
   
   cd ${DIR}/environment/ecsdemo-crystal
   kubectl delete -f kubernetes/service.yaml
   kubectl delete -f kubernetes/deployment.yaml
   
   cd ${DIR}/environment/ecsdemo-nodejs
   kubectl delete -f kubernetes/service.yaml
   kubectl delete -f kubernetes/deployment.yaml
   ```

2. 清理eks集群

   ```bash
   cd ${DIR}
   eksctl delete cluster -f chaos-toolkit-demo.yaml
   ```

   
