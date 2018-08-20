---
title: OpenAPI（Swagger）快速入门
date: 2018-08-13 22:00:48
? tags
toc: true
categories: ["doc", "swagger", "api"]
---

![banner](https://s1.ax1x.com/2018/08/13/Pg2TYD.png)

## Swagger 是什么

`Swagger`是使用`OpenAPI`规范（OAS）开发 API 的最广泛使用的工具生态系统。2015 年，`SmartBear Software`将`Swagger`规范捐赠给`Linux Foundation`，并将规范重命名为`OpenAPI`规范。 `SmartBear`还成为`OpenAPI Initiative（OAI）`的创始成员，该机构以开放和透明的方式管理`OAS`的发展。

> 简而言之 Swagger 包含了一套 API 规范，并且提供一系列的生态组件
> OpenAPI = 规范
> Swagger = 实现规范的组件

<!-- more -->

## Swagger 有什么用

`Swagger` 围绕着 `API` 诞生，它的用途也就是围绕着 `API`

- 作为 Rest API 的交互式文档
- 作为 Rest API 的形式化的接口描述
- 作为 Mock 服务的规范
- 作为调试文档生成测试页面 [Petstore](http://petstore.swagger.io/?_ga=2.171779168.1722816637.1534168865-221653252.1534168865#/)

## Swagger 怎么用

### YAML 编写

最为原始也最为可靠的方式就是人工编辑。`Swagger` 为我们提供了 `Swagger 编辑器`，在线环境 [Swagger Editor](https://editor.swagger.io)

#### OpenAPI 数据类型

| 常见数据类型 | type    | format    | 备注             |
| ------------ | ------- | --------- | ---------------- |
| integer      | integer | int32     | 32 位有符号数    |
| long         | integer | int64     | 64 位有符号数    |
| float        | number  | float     |                  |
| double       | number  | double    |                  |
| string       | string  |           |                  |
| byte         | string  | byte      | Base64 编码      |
| binary       | string  | binary    |                  |
| boolean      | boolean |           |                  |
| date         | string  | date      | RFC3339 格式     |
| dateTime     | string  | date-time | RFC3339 格式     |
| password     | string  | password  | 页面隐藏输入     |
| object       | object  |           | 由上面的数据构成 |

#### OpenAPI 根对象

| 字段名称     | 类型                          | 描述                   |
| ------------ | ----------------------------- | ---------------------- |
| openapi      | string                        | `必须`：版本号           |
| info         | Info Object                   | `必须`：API 定义的元数据 |
| servers      | [Server Object]               | 服务器的列表信息       |
| paths        | Paths Object                  | API 路由配置           |
| components   | Components Object             | 组件                   |
| security     | [Security Requirement Object] | 安全                   |
| tags         | [Tag Object]                  | 标签信息               |
| externalDocs | External Documentation Object | 额外文档               |


关于对象的定义在 [specification](https://swagger.io/specification/) 处进行查看


#### 编写API文档
`Swagger` 有两种编写格式，分别是 `Json` 和 `Yaml` 格式，个人更偏爱于 `Yaml` 格式，所以下文使用 `Yaml` 作为案例。
##### 定义接口元信息
```yaml
title: 这是标题 【必填】
description: 这里是一段描述
termsOfService: API可以测试地址
contact:
  name: 联系人
  url: http://www.example.com/support  联系人地址
  email: support@example.com  联系人邮箱
license:
  name: Apache 2.0 授权信息
  url: https://www.apache.org/licenses/LICENSE-2.0.html
version: 1.0.1  OpenAPI的版本号【必填】
```

##### 定义请求接口
```yaml
# 元信息略
/pets:  # 路径，如果需要 path value 可以使用形如 /pets/{petId}
  get:  # get 请求，同理还有 put，post，delete，options 等方法
    description: 这是一段描述信息
    responses:  # 响应部分定义
      '200':  # 200 的响应码
        description: 又是一段描述 
        content: #内容
          application/json:  # context-type
            schema:
              type: array  # 类型
              items:  # array 所包含的 元素
                $ref: '#/components/schemas/pet'  #每个元素是一个组件，下文详细说明
```
从一个简单的请求接口定义，我们就看出来，最为核心的`OPENAPI` 包含的是 `路径 /pets`，`请求方式 get`, `请求响应 responses`，但是好像缺少了点什么？对的，我们还不知道怎么去定义 `请求参数`。

###### 定义请求参数

```yaml
# 在 Header 中添加参数
name: token
in: header #这里是固定值
description: token to be passed as a header #描述信息
required: true # 是否必需
schema:
  type: array #类型
  items:
    type: integer
    format: int64

# 在 PATH 中添加参数
name: username
in: path #这里是固定值
description: username to fetch
required: true
schema:
  type: string

# 在 Query 中添加参数
name: id
in: query # 这里是固定值
description: ID of the object to fetch
required: false
schema:
  type: array
  items:
    type: string
style: form
explode: true
```

{% spoiler 整合在一起 %}
{% codeblock lang:yaml %}
/pets:  /# 路径，如果需要 path value 可以使用形如 /pets/{petId}
  get:  /# get 请求，同理还有 put，post，delete，options 等方法
    parameters:
      - name: token
        in: header #这里是固定值
        description: token to be passed as a header #描述信息
        required: true # 是否必需
        schema:
        type: array #类型
        items:
            type: integer
            format: int64
     -  name: username
        in: path #这里是固定值
        description: username to fetch
        required: true
        schema:
        type: string
     -  name: id
        in: query #这里是固定值
        description: ID of the object to fetch
        required: false
        schema:
        type: array
        items:
            type: string
        style: form
        explode: true
    description: 这是一段描述信息
    responses:  # 响应部分定义
      '200':  # 200 的响应码
        description: 又是一段描述 
        content: #内容
          application/json:  # context-type
            schema:
              type: array  # 类型
              items:  # array 所包含的 元素
                $ref: '#/components/schemas/pet'  #每个元素是一个定义，下文详细说明
{% endcodeblock %}
{% endspoiler %}

###### 定义请求体

我们使用 `requestBody` 作为请求体

```yaml
description: user to add to the system # 描述信息
content:  # 定义请求体
  'application/json': # context-type
    schema: # 请求的格式
      $ref: '#/components/schemas/User' # 组件
    examples: # 例子
      user:
        summary: User Example
        externalValue: 'http://foo.bar/examples/user-example.json'
  'application/xml': # context-type
    schema:
      $ref: '#/components/schemas/User'
    examples:
      user:
        summary: User Example in XML
        externalValue: 'http://foo.bar/examples/user-example.xml'
  'text/plain': # context-type
    examples:
      user:
        summary: User example in text plain format
        externalValue: 'http://foo.bar/examples/user-example.txt'
  '*/*':
    examples: # context-type
      user: 
        summary: User example in other format
        externalValue: 'http://foo.bar/examples/user-example.whatever'
```

{% spoiler 整合在一起 %}
{% codeblock lang:yaml %}
/pets:  /# 路径，如果需要 path value 可以使用形如 /pets/{petId}
  get:  /# get 请求，同理还有 put，post，delete，options 等方法
    parameters:
      - name: token
        in: header #这里是固定值
        description: token to be passed as a header #描述信息
        required: true # 是否必需
        schema:
        type: array #类型
        items:
            type: integer
            format: int64
     -  name: username
        in: path #这里是固定值
        description: username to fetch
        required: true
        schema:
        type: string
     -  name: id
        in: query #这里是固定值
        description: ID of the object to fetch
        required: false
        schema:
        type: array
        items:
            type: string
        style: form
        explode: true
    requestBody:
        description: user to add to the system # 描述信息
        content:  # 定义请求体
        'application/json': # context-type
            schema: # 请求的格式
            $ref: '#/components/schemas/User' # 组件
            examples: # 例子
            user:
                summary: User Example
                externalValue: 'http://foo.bar/examples/user-example.json'
        'application/xml': # context-type
            schema:
            $ref: '#/components/schemas/User'
            examples:
            user:
                summary: User Example in XML
                externalValue: 'http://foo.bar/examples/user-example.xml'
        'text/plain': # context-type
            examples:
            user:
                summary: User example in text plain format
                externalValue: 'http://foo.bar/examples/user-example.txt'
        '*/*':
            examples: # context-type
            user: 
                summary: User example in other format
                externalValue: 'http://foo.bar/examples/user-example.whatever'
    description: 这是一段描述信息
    responses:  # 响应部分定义
      '200':  # 200 的响应码
        description: 又是一段描述 
        content: #内容
          application/json:  # context-type
            schema:
              type: array  # 类型
              items:  # array 所包含的 元素
                $ref: '#/components/schemas/pet'  #每个元素是一个组件，下文详细说明
{% endcodeblock %}
{% endspoiler %}

直到这里，我们已经见过了  `Request部分 header,query,path,body` 如何组装。但是留在我们面前有一个小小的疑问，组件是什么？我们从下一章一窥究竟。

###### Schema 定义
我们知道我们的请求经常是特定的`Json`格式，这个 `Json` 会出现在我们的 `API` 定义的各个部分，`OPENAPI` 为了解决我们需要不断的重写相同的 `Json`定义，我们就有了`Schema`的概念(当然也包含Map定义等)。

```yaml
type: object # 类型
required: # 是否必需
- name
properties: # 内部属性定义
  name:
    type: string
  address:
    $ref: '#/components/schemas/Address'
  age:
    type: integer
    format: int32
    minimum: 0
```
{% spoiler 整合在一起 %}
{% codeblock lang:yaml %}
components:
  schemas:
    ErrorModel:
      type: object
      required:
      - message
      - code
      properties:
        message:
          type: string
        code:
          type: integer
          minimum: 100
          maximum: 600
    ExtendedErrorModel:
      allOf:
      - $ref: '#/components/schemas/ErrorModel'
      - type: object
        required:
        - rootCause
        properties:
          rootCause:
            type: string
{% endcodeblock %}
{% endspoiler %}

#### 小结
由于 `OpenApi` 的文档浩如烟海，这里就举了一个简单的例子，更多的时候还只能大家按图索骥的去查找自己想要的部分了。

### Swagger 插件

最近的使用方式是使用 `Swagger 插件` 从代码直接生成，举个最为简单的例子，我们在 `SpringBoot` 项目中增加一个依赖

```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.7.0</version>
</dependency>
```

并且在添加一个配置

```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
          .select()
          .apis(RequestHandlerSelectors.any())
          .paths(PathSelectors.any())
          .build();
    }
}
```

这个时候我们访问 `api-docs` 我们就可以获得文档

```bash
curl http://localhost:8080/spring-security-rest/api/v2/api-docs
{
    //略
}
```

当是我们有了文档，一点都不具有可读性。很高兴的官方也提供了 `UI`,继续添加

```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.7.0</version>
</dependency>
```

这个时候我们访问 `http://localhost:8080/your-app-root/swagger-ui.html` 我们就可以获得 页面了。
![pic](https://s1.ax1x.com/2018/08/13/PgRkXn.png)

## 生态工具

- [SwaggerHub](https://swagger.io/tools/)
   使用`OpenAPI`规范的设计和文档平台
- [Swagger Inspector](https://swagger.io/tools/swagger-inspector/)
   基于`OpenAPI`规范的测试平台
- [swagger-codegen](https://swagger.io/tools/swagger-codegen/)
    基于`OpenAPI`规范生成代码工具
- [swagger-editor](https://swagger.io/tools/swagger-editor/)
  `OpenAPI`的在线编辑器
- [swagger-ui](https://swagger.io/tools/swagger-ui/)
    `OpenAPI`的UI展示
- [ReDoc](https://github.com/Rebilly/ReDoc)
   更好看的`OpenAPI`的UI展示


## 参考资料

- [swagger-2-documentation-for-spring-rest-api](https://www.baeldung.com/swagger-2-documentation-for-spring-rest-api)
- [specification](https://swagger.io/specification/)