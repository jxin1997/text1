```markdown
# 项目实战：Jenkins-k8s-demo

## 项目介绍
该项目演示了使用 **Jenkins + Docker + Kubernetes + Harbor** 完整的 CI/CD 流水线实践。
需要环境：
- 代码仓库：GitLab 或 GitHub/Gitea  
- CI/CD 引擎：Jenkins  
- 镜像构建：Docker + Dockerfile  
- 私有镜像仓库：Harbor  
- 容器编排平台：Kubernetes（K8s）  
- 部署配置：Kubernetes Manifest 文件（YAML）

---

## 项目目录

```

hello-k8s-app/
├── app.py
├── requirements.txt
├── Dockerfile
├── Jenkinsfile
└── k8s/
├── deployment.yaml
└── service.yaml

````

---

## 文件内容

### app.py

```python
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def hello():
    hostname = os.uname().nodename
    return f"Hello, Kubernetes World! From Pod: {hostname}\n"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
````

### requirements.txt

```
Flask==2.3.3
```

### Dockerfile

```dockerfile
# 使用官方Python轻量级基础镜像
FROM python:3.9-slim

# 设置工作目录
WORKDIR /app

# 复制依赖文件并安装
COPY requirements.txt .
RUN pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

# 复制应用代码
COPY app.py .

# 暴露应用运行的端口
EXPOSE 5000

# 定义容器启动命令
CMD ["python", "app.py"]
```

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-k8s-app
  labels:
    app: hello-k8s-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-k8s-app
  template:
    metadata:
      labels:
        app: hello-k8s-app
    spec:
      containers:
      - name: hello-k8s-app
        image: harbor.your-domain.com/your-project/hello-k8s-app:latest
        ports:
        - containerPort: 5000
        imagePullPolicy: Always

---
apiVersion: v1
kind: Service
metadata:
  name: hello-k8s-app-service
spec:
  selector:
    app: hello-k8s-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  type: NodePort
```

### Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        REGISTRY = "registry.k8s.io"       // Harbor 地址
        PROJECT = "jenkins"
        APP_NAME = "hello-k8s-app"
        HARBOR_CREDENTIALS = credentials('12345678')  // Harbor 用户名密码
        KUBECONFIG_CREDENTIALS = credentials('kubeconfig-credentials') // Kubeconfig 文件
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master', url: 'http://192.168.10.20:32080/liupeng/edu.git'
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '12345678', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh """
                        echo "Logging into Harbor..."
                        docker login -u $HARBOR_USER -p $HARBOR_PASS $REGISTRY

                        echo "Building Docker image..."
                        docker build -t $REGISTRY/$PROJECT/$APP_NAME:BUILD-$BUILD_NUMBER ./hello-k8s-app

                        echo "Pushing Docker images..."
                        docker push $REGISTRY/$PROJECT/$APP_NAME:BUILD-$BUILD_NUMBER
                        docker tag $REGISTRY/$PROJECT/$APP_NAME:BUILD-$BUILD_NUMBER $REGISTRY/$PROJECT/$APP_NAME:latest
                        docker push $REGISTRY/$PROJECT/$APP_NAME:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG_FILE')]) {
                        sh """
                        mkdir -p $WORKSPACE/tmp_kube
                        cp $KUBECONFIG_FILE $WORKSPACE/tmp_kube/config
                        export KUBECONFIG=$WORKSPACE/tmp_kube/config

                        echo "Updating deployment.yaml with new image..."
                        sed -i 's|image:.*|image: $REGISTRY/$PROJECT/$APP_NAME:BUILD-$BUILD_NUMBER|' hello-k8s-app/k8s/deployment.yaml

                        echo "Applying deployment..."
                        kubectl apply -f hello-k8s-app/k8s/deployment.yaml
                        """
                    }
                }
            }
        }

        stage('Test Deployment') {
            steps {
                script {
                    sh """
                    export KUBECONFIG=$WORKSPACE/tmp_kube/config
                    NODE_PORT=\$(kubectl get svc hello-k8s-app-service -o jsonpath='{.spec.ports[0].nodePort}')
                    NODE_IP=\$(kubectl get node -o jsonpath='{.items[0].status.addresses[0].address}')
                    echo "Testing application at http://\$NODE_IP:\$NODE_PORT..."
                    curl --retry 10 --retry-delay 5 --retry-connrefused http://\$NODE_IP:\$NODE_PORT
                    """
                    echo "✅ 部署成功！应用访问地址：http://<NodeIP>:<NodePort>"
                }
            }
        }
    }

    post {
        always {
            echo "流水线执行完毕"
            sh "docker rmi $REGISTRY/$PROJECT/$APP_NAME:BUILD-$BUILD_NUMBER || true"
        }
    }
}
```

---

## 前提环境

1. Jenkins 已安装并运行，配置 Docker 和 Kubernetes 插件
2. 配置 Git 仓库、Harbor 仓库凭证
3. 配置 Kubernetes 集群凭证（通过 kubeconfig 文件）

---

## Jenkins 安装步骤

1. 安装 Java（推荐 Java 17）
```
yum install -y java-17-openjdk-devel
```
2. 下载 Jenkins RPM：

```bash
wget https://mirrors.aliyun.com/jenkins/redhat-stable/jenkins-2.516.2-1.1.noarch.rpm
rpm -ivh jenkins-2.516.2-1.1.noarch.rpm
```

3. 启动 Jenkins：

```bash
systemctl start jenkins
```

4. 访问 Jenkins：`http://192.168.10.20:8080/`

   * 默认用户名：`admin`
   * 密码：`/var/lib/jenkins/secrets/initialAdminPassword`

### 必装插件

* Docker Pipeline (`docker-pipeline`)
* Kubernetes CLI Plugin (`kubernetes-cli`)
* Git Plugin (`git`)

### 认证配置

1. Git 仓库凭证
2. Harbor 仓库凭证
3. Kubernetes 集群凭证（kubeconfig 文件）

### 创建流水线

1. 新建项目 → 流水线项目
2. 选择 SCM → Git → 选择 Jenkinsfile
3. 保存 → 运行流水线 → 查看执行结果与日志

---

## Webhook 配置

### 1. Jenkins 端

1. 安装 **Generic Webhook Trigger** 插件
2. Job 配置 → 触发器 → Generic Webhook Trigger
3. 设置 Token，例如 `mywebhooktoken`
4. Webhook URL：

```
http://192.168.10.32:8080/generic-webhook-trigger/invoke?token=mywebhooktoken
```

### 2. Gitea 仓库端

1. 仓库设置 → Webhooks → 新建 Webhook
2. 填写 Webhook URL：

```
http://192.168.10.32:8080/generic-webhook-trigger/invoke?token=mywebhooktoken
```

3. 保存，推送代码时 Jenkins 会自动触发构建

---

## 完整效果

1. 代码提交到 Git 仓库
2. Webhook 通知 Jenkins
3. Jenkins 拉取代码 → 构建 Docker 镜像 → 推送 Harbor
4. Jenkins 更新 Kubernetes Deployment → 部署应用
5. 测试 Pod 服务是否可访问
6. 清理临时镜像

