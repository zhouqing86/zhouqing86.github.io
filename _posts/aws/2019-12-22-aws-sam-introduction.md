---
layout: post
comments: false
categories: AWS
date:   2019-12-22 10:30:54
title: AWS SAM介绍
---

<div id="toc"></div>

> 即将换项目，据说新项目是servless架构。看有空提前准备点AWS Lambda知识

关于`SAM`的定义：

> AWS Serverless Application Model (SAM) is an open-source framework for building serverless applications

## 什么是Serverless Application

还是从AWS的文档中来看`serverless application`的定义如下：

```
A serverless application is a combination of Lambda functions, event sources, and other resources that work together to perform tasks. Note that a serverless application is more than just a Lambda function—it can include additional resources such as APIs, databases, and event source mappings.
```

其实关于`serverless`，意即其不需要`server`的，不过需要多说一句的是其实不需要申请任何AWS服务器的。


## 什么是AWS Lambda

AWS文档中对AWS Lambda开门见山的介绍：

```
AWS Lambda is a compute service that lets you run code without provisioning or managing servers.
```

为什么要使用AWS Lambda呢?AWS 文档也是相当的直接：

```
You pay only for the compute time you consume - there is no charge when your code is not running.
```

看样子用AWS Lambda是不需要服务器的，也是比较省钱的。

什么时候使用AWS Lambda呢？

- Run your code in response to events.

- Run your code in response to HTTP requests using Amazon API Gateway

- Invoke your code using API calls made using AWS SDKs

AWS文档也给出了一些`events`的场景：

- Changes to data in an Amazon S3 bucket

- Changes to data in an Amazon DynamoDB table

- Run your code in response to HTTP requests using Amazon API Gateway


## AWS SAM CLI

### 安装

SAM CLI依赖于docker，所以不管当前环境是Linux, Windows还是MacOS，首先要安装好docker。

笔者使用的是MacOS，且已经安装了`brew`神器，所以安装AWS CLI非常简单:

```
brew tap aws/tap
brew install aws-sam-cli
```

安装完成后，就可以直接使用`sam`命令了，如`sam -version`。

如果你希望在Windows环境安装SAM CLI，请参考官方文档 [Installing the AWS SAM CLI on Windows](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install-windows.html)

如果你希望在Linux环境安装SAM CLI，请参考官方文档 [Installing the AWS SAM CLI on Linux](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install-linux.html)


### 创建项目



### 测试和调试




## 参考文章

- [AWS Lambda Developer Guide](https://github.com/awsdocs/aws-lambda-developer-guide/blob/master/doc_source/index.md)

- [AWS Serverless Application Model (SAM) ](https://github.com/awslabs/serverless-application-model)

- [What Is the AWS Serverless Application Repository?](https://docs.aws.amazon.com/serverlessrepo/latest/devguide/what-is-serverlessrepo.html)

- [Using AWS SAM with the AWS Serverless Application Repository](https://docs.aws.amazon.com/serverlessrepo/latest/devguide/using-aws-sam.html)

- [Creating an Application with Continuous Delivery in the Lambda Console](https://github.com/awsdocs/aws-lambda-developer-guide/blob/master/doc_source/applications-tutorial.md)

- [Install the Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10)

- [Testing and Debugging Serverless Applications](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-test-and-debug.html)

- [AWS Toolkit for JetBrains](https://docs.aws.amazon.com/toolkit-for-jetbrains/latest/userguide/welcome.html)

- [Installing the AWS SAM CLI on Windows](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install-windows.html)

- [AWS Lambda](https://github.com/aws-samples/lambda-refarch-webapp)

- [What Is the AWS Serverless Application Repository?](https://docs.aws.amazon.com/serverlessrepo/latest/devguide/what-is-serverlessrepo.html)

- [Wild Rydes Serverless Workshops](https://github.com/aws-samples/aws-serverless-workshops)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
