---
title: graphql grpc in java world(1)
tags:
  - graphql
  - grpc
categories:
  - java
date: 2019-08-18 01:39:23
---

### 前言
graphql和grpc的protobuf的schema都是一个描述性文件，只是双方的具体作用有差别而已。在Java中使用schema first的`graphql-java-tools`无疑是graphql在java语言的最佳入门实践，那么问题就来啦！protobuf和graphql各自都有自己的类型系统，graphql因为会序列化为json，那么就要遵从java bean的规范（序列化框架要求），protobuf使用的builder构造对象，没有默认的构造方法。<br>
本文代码仓库地址:[https://github.com/silencecorner/graphql-grpc-exmaple/tree/0.2.0](https://github.com/silencecorner/graphql-grpc-exmaple/tree/0.2.0)，如果了解[graphql-java-kickstart](https://github.com/graphql-java-kickstart)的代码的话，可以直接查看源代码！
- `graphql-api` nodejs实现的网关
- `graphql-gateway-java` java实现的网关
- `post-api-java` post服务端微服务程序
- `protos` proto源文件
- `schema` graphql文件目录
- `vue-apollo-sample` 基于graphql规范的vue项目
### 优化思路
#### nodejs
因为在去年实践过一次，没有深入思考，写起来总感觉有一点别扭！所以最开始我的想法是改用nodejs来写去掉类型检查，也写过一个在[repo的graphql-api中](https://github.com/silencecorner/graphql-grpc-exmaple/tree/master/graphql-api)
#### converter
nodejs写起来挺简单的，但是java才是主要开发语言，所以又按照去年的那个套路实现了一次，按照converter的思路使用了[protobuf-converter](https://github.com/BAData/protobuf-converter)类库,优化了一下但是还是有一些不适。
#### jackson序列化框架
今天我就在想能不能jackson和protobuf之间做桥接一下，google搜索了果然已经有实现的[类库](https://github.com/HubSpot/jackson-datatype-protobuf)，终于不用再写一遍java model啦！
##### 删除代码
删除之前的inputs、types package，改用protobuf生成的代码，这里桥接要注入`ProtobufModule`，又想能不能直接使用返回`ListenableFuture`实例，通过查找资料可以实现。

##### 添加`GraphqlToolConfiguration.java`
```
@Configuration
public class GraphqlToolConfiguration {
	@Bean
	SchemaParserOptions options(){
		ObjectMapper objectMapper = Jackson2ObjectMapperBuilder.json()
			.modules(new ProtobufModule(), new Jdk8Module(), new KotlinModule(), new JavaTimeModule())
			.build();
		return SchemaParserOptions.newOptions()
			.genericWrappers(SchemaParserOptions.GenericWrapper
				.withTransformer(ListenableFuture.class,0,ListenableFuturesExtra::toCompletableFuture, type -> type))
			.objectMapperProvider(fieldDefinition -> objectMapper )
			.useDefaultGenericWrappers(true)
			.build();
	}
}
```
这样我们就可以使用protobuf生成的class、grpc直接返回的`ListenableFuture`，字段对应protobuf的[JsonName](https://developers.google.com/protocol-buffers/docs/style#message-and-field-names)，
##### schema.graphql
```graphql
scalar DateTime


# 作者
type Author{
  # unique id
  id: ID!
  # 名称
  name: String
}
# 添加作者参数
input AddAuthorRequest{
  # 名字
  name: String!
}

type Post {
  # id
  id: ID
  # 标题
  title: String
  # 内容
  body: String
  # 创建时间
  createdAt: DateTime
  # 文章作者
  author: Author
}

# 分页返回结果
type Posts {
  # 总数
  count: Int
  # 当前页
  page: Int
  # 条数
  limit: Int
  # 结点
  nodesList: [Post]
}

# 添加文章参数
input AddPostRequest {
  # 标题
  title: String
  # 内容
  body: String
}
# 分页参数
input ListPostRequest{
    # 第几页
    page: Int!
    # 获取条数
    limit: Int!
}
type Query {
  listPosts(request: ListPostRequest): Posts
}

type Mutation {
  addPost(request: AddPostRequest): Post
  # 新增作者
  addAuthor(request: AddAuthorRequest!): Author
}

schema {
  query: Query
  mutation: Mutation
}
```
##### Mutation.java
```java
@AllArgsConstructor
@Component
public class Mutation implements GraphQLMutationResolver {

    private final PostClient postClient;

    private final AuthorClient authorClient;

    public ListenableFuture<PostProto.Post> addPost(PostProto.AddPostRequest request){
		return postClient.addPost(request.toBuilder().setAuthorId(1).build());
    }

    public ListenableFuture<AuthorProto.Author> addAuthor(AuthorProto.AddAuthorRequest request){
    	  return authorClient.addAuthor(request);
    }
}
```
##### Query.java
```java
@AllArgsConstructor
@Component
public class Query implements GraphQLQueryResolver {
    private final PostClient postClient;
    public ListenableFuture<PostProto.Posts> listPosts(PostProto.ListPostRequest request){
        return postClient.listPost(request);
    }
}
```
##### PostResolver.java
```java
@Component
public class PostResolver implements GraphQLResolver<PostProto.Post> {

	@Autowired
	private AuthorClient authorClient;

	public ListenableFuture<AuthorProto.Author> author(PostProto.Post post){
		return authorClient.getAuthor(post.getAuthorId());
	}

}
```
#### PostsResolver.java
proto定义的是[nodes](https://github.com/silencecorner/graphql-grpc-exmaple/blob/master/protos/Post.proto#L26)字段，生成的java代码的get方法是`getNodesList`，此时对应的jackson的json字段就变成`nodesList`，我们的graphql中使用是`nodes`字段，按照`graphql-tools`的解析顺序
- com.bd.gateway.resolvers.post.PostsResolver.nodes(sample.PostProto$Posts)
- com.bd.gateway.resolvers.post.PostsResolver.getNodes(sample.PostProto$Posts)
- com.bd.gateway.resolvers.post.PostsResolver.nodes
- sample.PostProto$Posts.nodes()
- sample.PostProto$Posts.getNodes()
- sample.PostProto$Posts.nodes
  
添加如下代码就可以解决字段不一的问题啦！  
```
@Component
public class PostsResolver implements GraphQLResolver<PostProto.Posts> {

	public List<PostProto.Post> nodes(PostProto.Posts posts){
		return posts.getNodesList();
	}
}
```
##### PostClient.java
```
@Service
public class PostClient {
    @GrpcClient("post-grpc-server")
    private PostServiceGrpc.PostServiceFutureStub postServiceFutureStub;

    public ListenableFuture<PostProto.Post> addPost(sample.PostProto.AddPostRequest request){
        return postServiceFutureStub.addPost(request);
    }

    public ListenableFuture<PostProto.Posts> listPost(sample.PostProto.ListPostRequest request){
        return postServiceFutureStub.listPosts(request);
    }

}
```
##### AuthorClient.java
```java
@Service
public class AuthorClient {
    @GrpcClient("author-grpc-server")
    private AuthorServiceGrpc.AuthorServiceFutureStub authorServiceFutureStub;

    public ListenableFuture<AuthorProto.Author> addAuthor(AuthorProto.AddAuthorRequest request){
        return authorServiceFutureStub.addAuthor(request);
    }


    public ListenableFuture<AuthorProto.Author> getAuthor(Integer id){
        return authorServiceFutureStub.getAuthor(AuthorProto.GetAuthorRequest.newBuilder().setId(id).build());
    }
}
```
### new feature
[proto3](https://developers.google.com/protocol-buffers/docs/proto3)原生是不支持数据验证的，可能我们就要手写代码一个字段一个字段去做校验，项目中就会出现大量的丑陋到爆炸的代码。这里我找到一个protoc的[validate plugin](https://github.com/envoyproxy/protoc-gen-validate)，目前支持
- go
- gogo 
- cc for c++
- java
  
以目前情况来讲，不需要多语言调用，即使出现多语言调用的情况也可以，不影响正常调用，只是缺少验证而已，再不济也可以自己实现嘛！
#### 修改proto
```proto
import "validate/validate.proto";
message AddAuthorRequest{
    string name = 1 [(validate.rules).string = {min_len: 5, max_len: 10}];
}
```
#### 修改build.gradle
##### 添加必要依赖
```gradle
compile "io.envoyproxy.protoc-gen-validate:pgv-java-stub:${pgvVersion}"
compile "io.envoyproxy.protoc-gen-validate:pgv-java-grpc:${pgvVersion}"
```
##### 修改编译配置
```gradle
protobuf {
    // Configure the protoc executable
    protoc {
        artifact = "com.google.protobuf:protoc:${protocVersion}"
    }
    plugins {
        grpc {
            artifact = "io.grpc:protoc-gen-grpc-java:${grpcVersion}"
        }
        javapgv {
            artifact = "io.envoyproxy.protoc-gen-validate:protoc-gen-validate:${pgvVersion}"
        }
    }

    generateProtoTasks {
        all()*.plugins {
            javapgv {
                option "lang=java"
            }
            grpc {}
        }
    }
}
```

#### 添加客户端ValidatingClientInterceptor

```java
@Configuration
public class GlobalClientInterceptorConfiguration {

    @Bean
    public GlobalClientInterceptorConfigurer globalInterceptorConfigurerAdapter() {
	ValidatorIndex index = new ReflectiveValidatorIndex();
        return registry -> registry
			.addClientInterceptors(new LogGrpcInterceptor())
			.addClientInterceptors(new ValidatingClientInterceptor(index));
    }

}
```
#### 服务端添加ValidatingServerInterceptor
```
@Configuration
public class GlobalServerInterceptorConfiguration {

    @Bean
    public GlobalServerInterceptorConfigurer globalInterceptorConfigurerAdapter() {
		ValidatorIndex index = new ReflectiveValidatorIndex();
        return registry -> registry.addServerInterceptors(new ValidatingServerInterceptor(index));
    }

}
```
#### 测试
浏览器打开idea本地[http://localhost:8888/playground](http://localhost:8888/playground)<br>
浏览器打开docker本地[http://localhost:8000/playground](http://localhost:8888/playground)
```
mutation {
  addAuthor(request: { name: "" }) {
    id
    name
  }
}
```
因为名字验证规则为长度5~10，这里name值为空，执行返回结果
```
{
  "errors": [
    {
      "message": "INVALID_ARGUMENT: .sample.author.AddAuthorRequest.name: length must be 5 but got: 0 - Got \"\""
    }
  ],
  "extensions": {},
  "data": {
    "addAuthor": null
  }
}
```
生效，啦啦啦！
### 总结
介绍graphql、gprc in java world的一些问题，一些intergration的思路，新特性参数验证。

本文代码仓库地址:[https://github.com/silencecorner/graphql-grpc-exmaple/tree/0.2.0](https://github.com/silencecorner/graphql-grpc-exmaple/tree/0.2.0)
