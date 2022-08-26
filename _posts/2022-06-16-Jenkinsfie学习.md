---
layout: post
title: Jenkinsfile学习
date: 2022-06-16
tags: Jenkins
---


## 搭建Jenkins服务器

- 准备数据目录
```
mkdir /data/jenkins_data -p
chmod -R 777 /data/jenkins_data
```

- 启动Jenkins

```
docker run -d --name=jenkins --restart=always -e \
    JENKINS_PASSWORD=admin123 -e JENKINS_USERNAME=admin -e \
    JENKINS_HTTP_PORT_NUMBER=8080 -p 8080:8080 -p 50000:50000 -v \
    /data/jenkins_data:/bitnami/jenkins bitnami/jenkins:2.303.1-debian-10-r29
```
>  8080 端口为 Jenkins Web 界面的端口，50000 是 jnlp 使用的端口，后期 Jenkins Slave 需
要使用 50000 端口和 Jenkins 主节点通信

- 查看jenins日志

```
docker logs -f jenkins
... # 查看到这条日志说明 Jenkins 已完成启动
2022-06-16 03:50:38.985+0000 [id=21]    INFO    hudson.WebAppMain$3#run: Jenkins is fully up and running 
```

## 常用插件安装

- Git
- Git Parameter
- Git Pipeline for Blue Ocean
- GitLab
- Credentials
- Credentials Binding
- Blue Ocean
- Blue Ocean Pipeline Editor
- Blue Ocean Core JS
- Pipeline SCM API for Blue Ocean
- Dashboard for Blue Ocean
- Build With Parameters
- Dynamic Extended Choice Parameter Plug-In
- Dynamic Parameter Plug-in
- Extended Choice Parameter
- List Git Branches Parameter
- Pipeline
- Pipeline: Declarative
- Kubernetes
- Kubernetes CLI
- Kubernetes Credentials
- Image Tag Parameter
- Active Choices

> 替换为国内插件地址：`https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`

## 流水线

### 声明式流水线

在声明式流水线语法中，流水线过程定义在 Pipeline{}中，Pipeline 块定义了整个流水线中
完成的所有工作，比如：

``` 
pipeline {
  agent any
  stages {
    stage("Build") {
      steps {
        sh 'echo "Build"'
      }
    }
    stage("Test") {
      steps {
        sh 'echo "Test"'
      }
    }
    stage("Deploy") {
      steps{
        sh 'echo "Deploy"'
      }
    }
  }
}
```

#### agent

agent 表示整个流水线或特定阶段中的步骤和命令执行的位置，该部分必须在 pipeline 块的
顶层被定义，也可以在 stage 中再次定义，但是 stage 级别是可选的。

- any: 在任何可用的代理上执行流水线，配置语法

``` 
pipeline {
  agent any
}
```
- none: 表示该 Pipeline 脚本没有全局的 agent 配置。当顶层的 agent 配置为 none 时，
每个 stage 部分都需要包含它自己的 agent。配置语法：
``` 
pipeline {
  agent none
  stages {
    stage('Stage For Build'){
      agent any
    }
  }      
 }
```
- label：选择某个具体的节点执行 Pipeline 命令
``` 
pipeline {
  agent none
  stages {
    stage('Stage For Build'){
      agent { label 'slave-build-node' } 
    }
  } 
}
```

###  Stages

Stages 包含一个或多个 stage 指令，同时可以在 stage 中的 steps 块中定义真正执行的指令。

``` 
pipeline {
  agent any
  stages { 
    stage('Example') {
      steps {
        echo 'Hello World ${env.BUILD_ID}'
      }
    }
  } 
}
```

### Parameters

Parameters 提供了一个用户在触发流水线时应该提供的参数列表，这些用户指定参数的值可
以通过 params 对象提供给流水线的 step（步骤）。
目前支持的参数类型如下：
- string: 字符串类型
- text: 文本型参数，一般用于定义多行文本内容的变量
- booleanParam: 布尔型参数
- choice: 选择型参数，一般用于给定几个可选的值，然后选择其中一个进行赋值
- password：密码型变量，一般用于定义敏感型变量，在 Jenkins 控制台会输出为\*。
``` 
pipeline {
  agent any
  parameters {
    string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
    text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information about the person')
    booleanParam(name: 'TOGGLE', defaultValue: true, description: 'Toggle this value')
    choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'], description: 'Pick something')
    password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'Enter a password')
  }
  stages {
    stage('Example') {
      steps {
      echo "Hello ${params.PERSON}"
      echo "Biography: ${params.BIOGRAPHY}"
      echo "Toggle: ${params.TOGGLE}"
      echo "Choice: ${params.CHOICE}"
      echo "Password: ${params.PASSWORD}"
      }
    }
  }
}
```

### Triggers

在 Pipeline 中可以用 triggers 实现自动触发流水线执行任务，可以通过 Webhook、Cron、
pollSCM 和 upstream 等方式触发流水线。假如某个流水线构建的时间比较长，或者某个流水线需要定期在某个时间段执行构建，可以

- cron: 使用 cron 配置触发器定期执行，比如周一到周五每隔四个小时执行一次
``` 
pipeline {
  agent any
  triggers {
  cron('H */4 * * 1-5')
  }
  stages {
    stage('Example') {
      steps {
        echo 'Hello World'
      }
    }
  } 
}
```
- pollSCM：定义一个固定的间隔，在这个间隔中，Jenkins 会检查新的源代码更新。如果存在更改, 流水线就会被重新触发
``` 
pipeline {
  agent any
  triggers {
  pollSCM('H */4 * * 1-5')
  }
  stages {
    stage('Example') {
      steps {
        echo 'Hello World'
      }
    }
  } 
}
```
- Upstream: 可以根据上游 job 的执行结果决定是否触发该流水线
``` 
pipeline {
  agent any
  triggers {
  upstream(upstreamProjects: 'job1,job2', threshold: hudson.model.Result.SUCCESS)
  }
  stages {
    stage('Example') {
      steps {
        sh """
        echo "helloworld"
        sleep 5
        """
      }
    }
  } 
}
```

###  Input

input 字段可以实现在流水线中进行交互式操作，比如选择要部署的环境、是否继续执行某
个阶段等。
配置 Input 支持以下选项：
- message：必选，需要用户进行 input 的提示信息，比如：“是否发布到生产环境？”；
- id：可选，input 的标识符，默认为 stage 的名称；
- ok：可选，确认按钮的显示信息，比如：“确定”、“允许”；
- submitter：可选，允许提交 input 操作的用户或组的名称，如果为空，任何登录用户均
可提交 input； 
- parameters：提供一个参数列表供 input 使用。

假如需要配置一个提示消息为“还继续么”、确认按钮为“继续”、提供一个 PERSON 的变
量的参数，并且只能由登录用户为 alice 和 bob 提交的 input 流水线：

``` 
pipeline {
  agent any
  stages {
    stage('Example') {
      input {
        message "还继续么?"
        ok "继续"
        submitter "alice,bob"
        parameters {
          string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
        }
      }
      steps {
        echo "Hello, ${PERSON}, nice to meet you."
      }
    }
  } 
}
```


参考文档：https://www.jenkins.io/zh/doc/book/pipeline/syntax/
