## jenkins配置kubernetes插件

#### **一、配置插件**

![image-20200702160642028](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200702160642028.png)

**说明：**

```
1、Name 处默认为 kubernetes，也可以修改为其他名称，如果这里修改了，下边在执行 Job 时指定 podTemplate() 参数 cloud 为其对应名称，否则会找不到，cloud 默认值取：kubernetes

2、Kubernetes k8s内部地址地址为https://kubernetes.default.svc.cluster.local完整的DNS记录，<svc_name>.<namespace_name>.svc.cluster.local 的命名方式，或者写K8s的外部地址：https://<ClusterIP>:<Ports>

3、k8s名称空间：为部署jenkins所采用的名称空间

4、jenkins地址：使用 Jenkins Service 对应的 DNS 记录
```

#### **二、填写脚本测试**

通过 Kubernetes 安装 Jenkins Master 完毕并且已经配置好了连接，配置 Job 测试是否会根据配置的 Label 动态创建一个运行在 Docker Container 中的 Jenkins Slave 并注册到 Master 上，而且运行完 Job 后，Slave 会被注销并且 Docker Container 也会自动删除。

```groovy
def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: label, cloud: 'kubernetes') {
    node(label) {
        stage('Run shell') {
            sh 'sleep 130s'
            sh 'echo hello world.'
        }
    }
}
```

![这里写图片描述](https://img-blog.csdn.net/20180331115050146?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FpeGlhb3lhbmcxNjg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

#### 三、定义了一个 containerTemplate 模板

```groovy
def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: label, cloud: 'kubernetes', containers: [
    containerTemplate(name: 'maven', image: 'maven:3.3.9-jdk-8-alpine', ttyEnabled: true, command: 'cat'),
  ]) {

    node(label) {
        stage('Get a Maven Project') {
            git 'https://github.com/jenkinsci/kubernetes-plugin.git'
            container('maven') {
                stage('Build a Maven project') {
                    sh 'mvn -B clean install'
                }
            }
        }
    }
}
```

使用的 containers 定义了一个 containerTemplate 模板，指定名称为 maven 和使用的 Image，下边在执行 Stage 时，使用 container('maven'){...} 就可以指定在该容器模板里边执行相关操作了。比如，该示例会在 jenkins-slave 中执行 git clone 操作，然后进入到 maven 容器内执行 mvn -B clean install 编译操作。

![image-20200702162447782](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200702162447782.png)

说明：

Containers 下的 Name 字段的名字，这里要注意的是，<u>**如果 Name 配置为 jnlp，那么 Kubernetes 会用下边指定的 Docker Image 代替默认的 `jenkinsci/jnlp-slave` 镜像**</u>，否则，Kubernetes plugin 还是会用默认的 `jenkinsci/jnlp-slave` 镜像与 Jenkins Server 建立连接，即使我们指定其他 Docker Image。

#### 四、配置自定义jenkins-slave镜像

kubernetest plugin 默认提供的镜像 `jenkinsci/jnlp-slave` 可以完成一些基本的操作，但我们想执行maven编译或者docke命令或者kubectl命令时就有问题了。通过自己制作镜像来预装一些软件来既实现jenkin-slave功能又能满足其他的需求。

**192.168.31.11:8888/library/jenkins-slave:nlp 镜像集成maven、docker、docker-compose、kubectl**

```groovy
def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: label, cloud: 'kubernetes',containers: [
    containerTemplate(
        name: 'jnlp', 
        image: '192.168.31.11:8888/library/jenkins-slave:nlp', 
        alwaysPullImage: false, 
        args: '${computer.jnlpmac} ${computer.name}'),
  ]) {

    node(label) {
        stage('stage1') {
            stage('Show Maven version') {
                sh 'mvn -version'
                sh 'sleep 10'
            }
        }
    }
}
```

**说明：**

里 containerTemplate 的 name 属性必须叫 jnlp，Kubernetes 才能用自定义 images 指定的镜像替换默认的 jenkinsci/jnlp-slave 镜像。此外，args 参数传递两个 jenkins-slave 运行需要的参数。还有一点就是这里并不需要指定 container('jnlp'){...} 了，因为它被 Kubernetes 指定了要被执行的容器，所以直接执行 Stage 就可以了。