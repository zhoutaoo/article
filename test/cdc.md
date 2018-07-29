微服务之契约测试 Spring Cloud Contract
=========

## CDC消费者驱动的契约测试

> *消费者驱动的契约测试（Consumer-Driven Contracts，简称CDC），是指从消费者业务实现的角度出发，驱动出契约，再基于契约，对提供者验证的一种测试方式。*

### **为什么要做契约测试**
 
  假设我们有一个由多个微服务组成的系统：如图

![微服务](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/LsNc01I3kx62AaMvwAoHaapibODUna9ScIp7kvNiaSVnnKJ59CzTRE39QKrTQj1CQPXTqGzTmu0u61ZwqW8IyFZA/640?wx_fmt=png&wxfrom=5&wx_lazy=1)

如果我们想测试应用v1，我们可以做以下两件事之一：

* 部署所有微服务并执行端到端测试。
* 在单元/集成测试中模拟其他微服务。

两者都有其优点，但也有很多缺点。

**部署所有微服务并执行端到端测试**

优点：

* 模拟生产。
* 测试服务之间的真实通信。

缺点：

* 要测试一个微服务，我们必须部署6个微服务，几个数据库等。
* 运行时间很长，稳定性差，容易失败。
* 非常难以调试，依赖服务不受控制。

**在单元/集成测试中模拟其他微服务**

优点：

* 非常快速的反馈，简单易用。
* 他们没有基础设施要求,如DB，网络等。

缺点：

* 模拟不够真实。
* 部分场景测试不到。

### **使用Spring Cloud Contract后**

如下：测试v1就不用启动其它服务了

![stub的测试](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/LsNc01I3kx62AaMvwAoHaapibODUna9Sc2MYSBeV55UkYNib9naJaGpEoshgjbb764n7Lia7nwkGFyfaAN7VpQzcw/640?wx_fmt=png)

## 契约测试（Contract）

## 契约测试步骤
Spring Cloud Contract契约测试大概分三个步骤
1. producer提供服务的定好服务接口（即契约）
2. 生成stub，并共享给消费方，可通过mvn install到maven库中
3. consumer消费方引用契约服务，进行集成测试

## Server/Producer 服务提供端

### 构架引入
* 在pom.xml中加入jar包依赖，放入dependencies中

```
<!--契约测试服务提供端依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-verifier</artifactId>
    <scope>test</scope>
</dependency>
```
* 配置插件，放入plugins中

```
<plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <!-- Don't forget about this value !! -->
    <extensions>true</extensions>
    <configuration>
        <!-- MvcMockTest为生成本地测试案例的基类 -->
        <baseClassForTests>com.springboot.services.producer.MvcMockTest</baseClassForTests>
    </configuration>
</plugin>
```

* spring-cloud-contract-maven-plugin的使用介绍

`spring-cloud-contract:help`：帮助

`spring-cloud-contract:convert`：在target/stubs下，根据将契约生成mapping文件，用于打包jar文件，提供http服务，供consumer使用

`spring-cloud-contract:generateStubs`：生成stubs的jar包，用于分享给consumer使用

`spring-cloud-contract:generateTests`：基于contract生成服务契约的测试案例，服务实现了契约后，保证实现与契约一致

`spring-cloud-contract:run`：启动契约服务，将契约暴露为http server服务

`spring-cloud-contract:pushStubsToScm`：将契约放置在scm中管理

### 需求

* 假设有需求 producer服务需要提供一个对外的接口GET请求的接口，并且接收一个name的参数，如
GET /hello?name=zhangsan，要求返回`{
	"code": "000000",
	"mesg": "处理成功"
}` 

#### 1. 写Controller
注：对象Result对象封装了
`{
	"code": "000000",
	"mesg": "处理成功"
}` 数据

```
package com.springboot.services.producer.rest;
import com.springboot.cloud.common.core.entity.vo.Result;
import org.springframework.web.bind.annotation.*;
import static org.apache.commons.lang.RandomStringUtils.randomNumeric;

@RestController
public class HelloController {

    @RequestMapping(method = RequestMethod.GET, value = "/hello")
    public Result world(@RequestParam String name) {
        return Result.success(name);
    }
}
```
#### 2. 编写契约

在src/test/resources/contracts/HelloController.groovy 中增加契约文件（可以有多种格式如groovy 、yaml、，这里采用groovy）
契约书写如下：

```
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    request {
        method 'GET'
        url('/hello') {
            queryParameters {
                parameter("name", "zhangsan")
            }
        }

    }
    response {
        status 200
        body("""
  {
    "code": "000000",
    "mesg": "处理成功"
  }
  """)
        headers {
            header('Content-Type': 'application/json;charset=UTF-8')
        }
    }
}
```

#### 3.生成stub jar文件

* 执行：`mvn spring-cloud-contract:convert`命令，会在target/stubs下生成相关文件

![目录结构](http://baidu.com/index.ico)

生成的mapping文件，样式如下

```
{
  "id" : "62db0b7f-72de-4c03-8e38-6874d4b433ab",
  "request" : {
    "urlPath" : "/hello",
    "method" : "GET",
    "queryParameters" : {
      "name" : {
        "equalTo" : "zhangsan"
      }
    }
  },
  "response" : {
    "status" : 200,
    "body" : "{\"code\":\"000000\",\"mesg\":\"处理成功\"}",
    "headers" : {
      "Content-Type" : "application/json;charset=UTF-8"
    },
    "transformers" : [ "response-template" ]
  },
  "uuid" : "62db0b7f-72de-4c03-8e38-6874d4b433ab"
}
```

* 执行：`mvn spring-cloud-contract:generateStubs`命令，会在target下生成stubs的jar包
如：producer-0.0.1-SNAPSHOT-stubs.jar

#### 4.安装stub到maven库中

当然也可将plugin 绑定到相关的phase上自动安装到maven库中
例子：
`mvn install:install-file -DgroupId=com.springboot.cloud -DartifactId=producer -Dversion=0.0.1-SNAPSHOT -Dpackaging=jar -Dclassifier=stubs  -Dfile=producer-0.0.1-SNAPSHOT-stubs.jar`

**以上，将stub jar包分享给consumer，对方就要以在集成测试案例中使用了**


#### 5.接口实现并检验是否符合契约

* 提供一个测试基类（主要用于按照契约对接口生成测试案例，检验接口是否按契约实现了）

```
package com.springboot.services.producer;

import com.springboot.services.producer.rest.HelloController;
import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.Before;
import org.junit.Test;

public class MvcMockTest {
    @Before
    public void setup() {
        RestAssuredMockMvc.standaloneSetup(new HelloController());
    }
}
```

* 执行：`mvn spring-cloud-contract:generateTests`命令，会在target/generated-test-sources/contracts目录下根据契约生成测试案例，用于服务提供方最后检验是否符合契约。

例子：

```
package com.springboot.services.producer;

import com.jayway.jsonpath.DocumentContext;
import com.jayway.jsonpath.JsonPath;
import com.springboot.services.producer.MvcMockTest;
import io.restassured.module.mockmvc.specification.MockMvcRequestSpecification;
import io.restassured.response.ResponseOptions;
import org.junit.Test;

import static com.toomuchcoding.jsonassert.JsonAssertion.assertThatJson;
import static io.restassured.module.mockmvc.RestAssuredMockMvc.*;
import static org.springframework.cloud.contract.verifier.assertion.SpringCloudContractAssertions.assertThat;

public class ContractVerifierTest extends MvcMockTest {

	@Test
	public void validate_helloController() throws Exception {
		// given:
		MockMvcRequestSpecification request = given();

		// when:
		ResponseOptions response = given().spec(request)
					.queryParam("name","zhangsan")
					.get("/hello");

		// then:
		assertThat(response.statusCode()).isEqualTo(200);
		assertThat(response.header("Content-Type")).isEqualTo("application/json;charset=UTF-8");
		// and:
		DocumentContext parsedJson = JsonPath.parse(response.getBody().asString());		assertThatJson(parsedJson).field("['code']").isEqualTo("000000");
        assertThatJson(parsedJson).field("['mesg']").isEqualTo("\u5904\u7406\u6210\u529F");
	}
}
```

## Client/Consumer 服务调用端

### 构架引入
* jar包依赖，放入dependencies中

```
<!--契约测试服务提供端依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
</dependency>
```

### 调用方代码实现

**假如消费方有接口/classes?name=xxx，该接口调用了producer服务的街道口/hello?name=xxx，此时**

* ClassController如下，调用ClassService

```
package com.springboot.feign.rest;
import com.springboot.cloud.common.core.entity.vo.Result;
import com.springboot.feign.service.ClassService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import java.util.Map;

@RestController
public class ClassController {

    @Autowired
    private ClassService classService;

    @GetMapping("/classes")
    public Result hello(@RequestParam String name) {
        return classService.users(name);
    }
}
```

* ClassService如下，通过feigin调用producer服务

```
package com.springboot.feign.service;
import com.springboot.cloud.common.core.entity.vo.Result;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import java.util.Map;

@FeignClient(name = "producer")
public interface ClassService {

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    Result users(@RequestParam("name") String name);
}
```
### 测试案例编写

* 集成测试案例如下：

```
package com.springboot.feign.rest;

import org.hamcrest.core.Is;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
import org.springframework.cloud.contract.stubrunner.spring.StubRunnerProperties;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;

@RunWith(SpringRunner.class)
//springboot的测试启动类，需要依赖spring-boot-test库
@SpringBootTest
//初使化测试测试配置，测试controller需要
@AutoConfigureMockMvc
//启动契约服务，模拟produer提供服务
@AutoConfigureStubRunner(ids = {"com.springboot.cloud:producer:+:stubs:8080"}, stubsMode = StubRunnerProperties.StubsMode.LOCAL)
public class ClassControllerTest {

    @Autowired
    private MockMvc mvc;

    @Test
    public void testMethod() throws Exception {
        mvc.perform(MockMvcRequestBuilders.get("/classes").param("name", "zhangsan"))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("code", Is.is("000000")));
    }
}
```

* 简介@AutoConfigureStubRunner

`@AutoConfigureStubRunner(ids = {"com.springboot.cloud:producer:+:stubs:8080"}, stubsMode = StubRunnerProperties.StubsMode.LOCAL)`

**ids格式如下：**

> groupId:artifactId:version:classifier:port

**`stubsMode = StubRunnerProperties.StubsMode.LOCAL`**

表示从哪里加载stub的jar包，有3种：

CLASSPATH：从classpath中找jar包，默认

LOCAL：从本地maven库中找

REMOTE：从远程服务器中下载，需要配合git,并将repositoryRoot设定值，如： 
```
@AutoConfigureStubRunner(
    stubsMode="REMOTE",
    repositoryRoot="git://https://github.com/spring-cloud-samples/spring-cloud-contract-nodejs-contracts-git.git",
    ids="com.example:bookstore:0.0.1.RELEASE"
)
```


### 测试执行

`mvn test`

**完整的例子，请查看github**

[producer](https://github.com/zhoutaoo/SpringCloud/tree/master/services/producer) 和
[consumer-feign](https://github.com/zhoutaoo/SpringCloud/tree/master/services/consumer-feign)

谢谢



