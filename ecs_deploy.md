# AWS ECS 部署手顺

### 1. 前提准备
在开始部署之前，你需要准备以下内容：
* AWS 账号（IAM 用户最好有 AdministratorAccess 权限，或者至少有 ECS、ECR、VPC、IAM、CloudWatch、ALB 权限）
* 安装并配置 AWS CLI (aws configure)
* 安装 Docker（用于构建镜像）

### 2. 构建 Docker 镜像 & 推送到 Amazon ECR
ECS 不直接存储镜像，而是从 ECR (Elastic Container Registry) 拉取。 
#### 2.1 创建 ECR 仓库
  ```bash
  aws ecr create-repository --repository-name <focason-lab/eureka>
  ```
#### 2.2 Authenticate Docker to ECR:
  ```shell
  aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com
  ```
#### 2.3 Build, tag, and push image:
  ```bash
  docker build -t focason-lab/eureka:latest .
  docker tag focason-lab/eureka:latest <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com/focason-lab/eureka:latest
  docker push <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com/focason-lab/eureka:latest
  ```

### 3. 配置网络（VPC, Subnet, Security Group）
ECS 部署依赖 AWS 网络： 
#### 3.1 创建或使用已有 VPC
#### 3.2 至少两个 Subnet（推荐公有子网 + 私有子网）
#### 3.3 创建 Security Group
  * 允许前端端口（80/443）
  * 允许微服务之间通信（例如 Gateway → Auth 服务 8085 端口）
#### 3.4 创建 Load Balancer (ALB)（用于对外暴露服务，支持 HTTPS）

### 4. 配置 ECS Cluster
* AWS 控制台 → ECS → Create Cluster → Networking only (Fargate) 方式创建。
* 或用命令行直接创建
  ```bash
  aws ecs create-cluster --cluster-name <focason-lab-cluster>
  ```

### 5. 创建 IAM 角色
ECS 需要角色来访问 ECR、CloudWatch 等：
* `ecsTaskExecutionRole`
  * 附加策略： `AmazonECSTaskExecutionRolePolicy`
  * 权限：允许 ECS 任务从 ECR 拉取镜像，向 CloudWatch 写日志。

### 6. 定义 Task Definition
ECS 的任务定义描述 容器如何运行。  
  * 参考 `./deploy/focason-lab-eureka-server-task.json`
  * 注册任务定义：
    ```bash
    aws ecs register-task-definition --cli-input-json file://auth-service-task.json
    ```

### 7. 创建 ECS Service
一个 Service = 长期运行的任务实例，支持自动扩缩容。
  * 创建Service
    ```bash
    aws ecs create-service \
    --cluster focason-cluster \
    --service-name auth-service \
    --task-definition focason-auth-service \
    --desired-count 1 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=ENABLED}" \
    --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:ap-northeast-1:xxx:targetgroup/auth-service-tg/xxx,containerName=auth-service,containerPort=8085"
    ```
    ※注意
    * 前端服务（Nuxt）可直接挂到 ALB，暴露 80/443。
    * Gateway 和 Auth 服务可以放在私有子网，只允许 ALB 或内部服务访问。

### 8. 监控与日志
* CloudWatch Logs：收集应用日志。
* CloudWatch Metrics：ECS CPU/内存指标，结合 Auto Scaling 自动扩缩容。
* X-Ray：可选，做分布式追踪。





