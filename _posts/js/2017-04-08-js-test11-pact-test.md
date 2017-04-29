---
layout: post
comments: false
categories: "测试JS"
date:   2017-4-8 18:30:54
title: JS测试之Pact测试
---

<div id="toc"></div>
<br />
Pact在英文上的解释是"a formal agreement between individuals or parties", 其实协议、约定或条约。

Pact测试又名契约测试，是在消费者服务于生产者服务之间的 `消费者驱动测试`。 本文后续内容将只使用 `契约测试` 来表示Pact测试。

## 不平等合约

从 `消费者驱动测试` 我们能感觉到契约测试中两个角色消费者服务于生产者服务之间的不平等。

站在吃瓜群众的角度，什么是平等合约和不平等合约呢，我们拿一个现实中的例子来说明，生产者是服装定制生产企业(代号P)，消费者是每年都需要批量定制工装的某银行(代号C)，P和C每年签订合约。

1. 第一年，C统计今年工装的需求（款式、颜色、数量、预算等），并附加很多苛刻奇葩的要求，并拟定了合约，P为了从其他竞争对手中抢到C的订单，P接受了这个挣不到钱的不平等合约。

2. 第二年，由于去年C对P提供的工装非常满意，P也在去年与C的合作中增进了关系，P与C1请了第三方会计事务所来修订
第一年的合约，达成了一份相对公平的合约。

契约测试的过程映射到的是第一年的不平等条约的签订过程，在IT系统中，P是生产者端，C是消费者端。概括起来步骤如下:

1. 消费者端统计需求，拟定期望从生产者端获取的结果，制作契约。

2. 消费者将契约存储在一个公共区域以便生产者读取。

3. 生产者拿取契约并做验证，确定自己能够为C生产出契约规定的结果。

<br />

## 解决了什么问题

在复杂的IT系统中，特别是微服务架构的IT系统中，存在大量的服务，服务于服务之间的关系可谓错综复杂。我们经常会遇到如下问题：

1. 一个API上的小改动，导致下游的一个或多个消费者服务挂掉。

2. 做了版本控制的API版本都到了V10了，还得维护V1-V10各个版本的endpoint以及详尽的各个版本API文档。

3. 遗留的系统的系统架构图貌似过时了，依赖关系剪不断理还乱。

4. 删除某个API，不确定谁还在用它。

契约测试试图基于契约快速反馈，解决上述的问题：

1. 生产者端做契约的验证，即便是一次小的改动，也会验证是否会针对不同的消费者给出规定的结果。

2. 提取一层契约管理者，存储和展示契约，基于契约描绘生产者们与消费者们之间的关系。

3. 消费者端需求变化，如果修改了需求，就会将修改的契约提交给契约管理者。

4. 生产者端每次做契约验证之前，都是从契约管理者手中拉取最新的契约来验证。

5. 生产者端可以测试多个消费者端。

<br />

## 契约测试前要考虑的问题

基于契约测试的理论框架，如果我们要引入契约测试，需要解决的问题：

1. 开发环境中搭建契约管理者环境，展示指定上传的契约的文件命名规则等。

2. 不同的服务可能使用不同的技术栈(Ruby, JS, Java, Python等)，契约的格式需要统一且互相之间易于解析。

3. 将Pact测试集成到持续集成管道中，便于自动化验证。

<br />

## 实践入门

契约测试的例子相对各种语言的HelloWorld来说，是比较复杂的，为了使得例子尽量简单易懂，我们先做一些说明：

1. 本地做Pact测试可以不需要搭建契约管理者环境，消费者端生成的契约拷贝到生产者端进行验证即可。

2. 消费者端持有用户的ID， GET '/user/:id' 需要获取用户的姓名。

3. 生产者端判断请求是 GET '/user/:id', 则返回用户的ID以及用户名。

{% include image.html url="/static/img/js/user-service-pact.png" description="契约" height="200px" inline="true" %}

### JS例子

#### 消费者端测试

1. 参考[JS测试之Mocha](/2017/03/26/js-test4-mocha/)中的 `Node.js + mocha + chai`章节创建本地测试环境。

2. `npm install -D pact`安装npm的pact包 [pact](https://www.npmjs.com/package/pact)。

3. 创建test/consumer/consumer-test.js文件:

    1) 创建一个Pact对象，其表示依赖的一个生产者端

    ```
    const provider = pact({
      consumer: 'TodoApp',
      provider: 'TodoService',
      port: 8002,
      log: path.resolve(process.cwd(), 'logs', 'pact.log'),
      dir: path.resolve(process.cwd(), 'pacts'),
      logLevel: 'INFO',
      spec: 2
    });
    ```

    2) `provider.setup()` 启动mock server来mock生产者服务

    3）定义消费者与生产者相互交互的内容，与前一步结合

    ```
    provider.setup()
        .then(() => {
          provider.addInteraction({
            state: 'have a matched user',
            uponReceiving: 'a request for get user',
            withRequest: {
              method: 'GET',
              path: '/user/1'
            },
            willRespondWith: {
              status: 200,
              headers: { 'Content-Type': 'application/json' },
              body: {
                id: 1,
                name: 'God'
              }
            }
          })
        })
        .then(() => done())
    ```

    4）测试代码中需要有逻辑请求mock的生产者服务

    ```
    it('should response with user with id and name', (done) => {
      request.get('http://localhost:8002/user/1')
        .then((response) => {
          const user = response.body;
          expect(user.name).to.equal('God');
          provider.verify();
          done();
        })
        .catch((e) => {
          console.log('error', e);
          done(e);
        });
    });
    ```

    5) 将契约写到文件中，关闭mock的生产者端

    ```
    after(() => {
      provider.finalize();
    });
    ```

    6) 修改package.json `test`脚本为 `mocha --timeout 15000`，而后执行 `npm test`

    7）`pacts` 目录下查看将生成`todoapp-todoservice.json`文件

    ```
    {
      "consumer": {
        "name": "TodoApp"
      },
      "provider": {
        "name": "TodoService"
      },
      "interactions": [
        {
          "description": "a request for get user",
          "provider_state": "have a matched user",
          "request": {
            "method": "GET",
            "path": "/user/1"
          },
          "response": {
            "status": 200,
            "headers": {
              "Content-Type": "application/json"
            },
            "body": {
              "id": 1,
              "name": "God"
            }
          }
        }
      ],
      "metadata": {
        "pactSpecificationVersion": "2.0.0"
      }
    }
    ```

#### 生产者端测试

生产者端的流程往往是从下层API，数据库或其他持久化存储中搜索/读取数据，在生产者中做数据加工后输出给消费者端。

生产者端拿到契约后，需要验证的往往是数据加工到返回数据的过程。所以往往需要mock底层数据。

不过下面的例子为了简便，没只是mock了返回数据， 也没有考虑底层数据，也没有考虑数据加工的逻辑。

在消费者测试的测试基础设施基础上，创建test/provider/provider-test.js:

1. 启动本地的生产者端server，暴露出`/user/:id`接口，直接返回了mock的数据。

    ```
    const express = require('express');
    const cors = require('cors');
    const bodyParser = require('body-parser');


    const server = express();
    server.use(bodyParser.json());

    server.use((req, res, next) => {
      res.header('Content-Type', 'application/json');
      next();
    });

    server.get('/user/:id', (req, res) => {
      res.end(JSON.stringify({
        id: 1,
        name: 'God1'
      }));
    });
    ```

2. 创建辅助endpoint `/states` 暴露api接口当前的状态(状态在消费者端生成的契约中是 `provider_state` 下的值 )。

    ```
    server.get('/states', (req, res) => {
      res.json({
        "TodoApp": ['have a matched user']
      });
    });
    ```

3. 创建辅助endpoint `/setup`, 其往往根据state的不同来初始化不同底层数据，或者mock底层数据，在这个例子我们没有考虑mock底层数据，所以/setup只是打印了状态。

    ```
    server.post('/setup', (req, res) => {
      const state = req.body.state
      console.log('state:', state);
      res.end()
    });
    ```

4. 启动server，暴露端口8081

    ```
    server.listen(8081, () => {
      console.log('User Service listening on http://localhost:8081')
    });
    ```

5. 添加测试用例来验证生产者满足消费者的需求

    ```
    describe('Pact Verification', () => {
      it('should validate the expectations of Matching Service', () => {

        const opts = {
          providerBaseUrl: 'http://localhost:8081',
          providerStatesUrl: 'http://localhost:8081/states',
          providerStatesSetupUrl: 'http://localhost:8081/setup',
          pactUrls: [path.resolve(process.cwd(), './pacts/todoapp-todoservice.json')]
        }

        return verifier.verifyProvider(opts)
          .then(output => {
            console.log('Pact Verification Complete!')
            console.log(output)
          });
      });
    });
    ```

    1) opts定义的providerStatesUrl为上面定义的获取生产者的states，providerStatesSetupUrl上定义setup的url。

    2）使用 `require('pact').Verifier.verifyProvider` 来验证契约。

### Java例子

#### 消费者端测试

#### 生产者端测试

## Pact Broker

## 契约测试存在的问题

1. 生产者端的开发/维护人员的每次提交都对Pact测试负责，即便是消费者端修改导致Pact挂掉。

2. 消费者契约变化后提交给契约管理者与生产者提取契约验证两个行为之间存在时间差。

3. 目前契约测试的环境只支持HTTP协议。

4. 生产者端往往需要mock底层数据以使得其能满足consumer的结果要求。

## 与其他测试的区别于关系
### 契约测试不是集成测试


### 契约不是E2E测试

### GraphQL与契约测试

### BFF与契约测试

### 参考资料

- [nodejs pact](https://www.npmjs.com/package/pact)


<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
