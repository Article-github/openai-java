![Maven Central](https://img.shields.io/maven-central/v/com.theokanning.openai-gpt3-java/client?color=blue)

> ⚠️OpenAI has deprecated all Engine-based APIs. See [Deprecated Endpoints](https://github.com/TheoKanning/openai-java#deprecated-endpoints) below for more info.
# OpenAI-Java

Java库，用于使用OpenAI的GPT API。支持GPT-3、ChatGPT和GPT-4。

包括以下组件：

- `api`：用于GPT API的请求/响应POJO。
- `client`：用于GPT端点的基本Retrofit客户端，包括`api`模块。
- `service`：一个基本的服务类，用于创建和调用客户端。这是入门的最简单方式。

以及一个使用该服务的示例项目。

## 支持的API

- [模型](https://platform.openai.com/docs/api-reference/models)
- [完成](https://platform.openai.com/docs/api-reference/completions)
- [聊天完成](https://platform.openai.com/docs/api-reference/chat/create)
- [编辑](https://platform.openai.com/docs/api-reference/edits)
- [嵌入](https://platform.openai.com/docs/api-reference/embeddings)
- [音频](https://platform.openai.com/docs/api-reference/audio)
- [文件](https://platform.openai.com/docs/api-reference/files)
- [微调](https://platform.openai.com/docs/api-reference/fine-tuning)
- [图像](https://platform.openai.com/docs/api-reference/images)
- [审查](https://platform.openai.com/docs/api-reference/moderations)
- [助手](https://platform.openai.com/docs/api-reference/assistants)

#### OpenAI已废弃

- [引擎](https://platform.openai.com/docs/api-reference/engines)
- [传统微调](https://platform.openai.com/docs/guides/legacy-fine-tuning)

## 导入

### Gradle

`implementation 'com.theokanning.openai-gpt3-java:<api|client|service>:<version>'`

### Maven

```xml
   <dependency>
    <groupId>com.theokanning.openai-gpt3-java</groupId>
    <artifactId>{api|client|service}</artifactId>
    <version>version</version>       
   </dependency>
```

## 用法

### 仅数据类

如果要创建自己的客户端，只需从`api`模块导入POJO。您的客户端将需要使用蛇形命名法与OpenAI API配合使用。

### Retrofit客户端

如果使用retrofit，可以导入`client`模块并使用[OpenAiApi](client/src/main/java/com/theokanning/openai/OpenAiApi.java)。  
您将需要将您的身份验证令牌添加为头部（参见[AuthenticationInterceptor](client/src/main/java/com/theokanning/openai/AuthenticationInterceptor.java)）并设置您的转换器工厂以使用蛇形命名法并且仅包含非空字段。

### OpenAiService

如果您正在寻找最快的解决方案，请导入`service`模块并使用[OpenAiService](service/src/main/java/com/theokanning/openai/service/OpenAiService.java)。  

> ⚠️客户端模块中的OpenAiService已弃用，请切换到服务模块中的新版本。

```java
OpenAiService service = new OpenAiService("your_token");
CompletionRequest completionRequest = CompletionRequest.builder()
        .prompt("Somebody once told me the world is gonna roll me")
        .model("babbage-002"")
        .echo(true)
        .build();
service.createCompletion(completionRequest).getChoices().forEach(System.out::println);
```

### 自定义OpenAiService

如果需要自定义OpenAiService，请创建自己的Retrofit客户端并将其传递给构造函数。
例如，按照以下步骤添加请求日志记录（在添加日志记录gradle依赖项后）：

```java
ObjectMapper mapper = defaultObjectMapper();
OkHttpClient client = defaultClient(token, timeout)
        .newBuilder()
        .interceptor(HttpLoggingInterceptor())
        .build();
Retrofit retrofit = defaultRetrofit(client, mapper);

OpenAiApi api = retrofit.create(OpenAiApi.class);
OpenAiService service = new OpenAiService(api);
```

### 添加代理

要使用代理，请修改OkHttp客户端如下所示：

```java
ObjectMapper mapper = defaultObjectMapper();
Proxy proxy = new Proxy(Proxy.Type.HTTP, new InetSocketAddress(host, port));
OkHttpClient client = defaultClient(token, timeout)
        .newBuilder()
        .proxy(proxy)
        .build();
Retrofit retrofit = defaultRetrofit(client, mapper);
OpenAiApi api = retrofit.create(OpenAiApi.class);
OpenAiService service = new OpenAiService(api);
```

### 函数

您可以轻松创建自己的函数，并使用ChatFunction类定义它们的执行器，以及您的自定义类，以定义它们的可用参数。您还可以轻松处理函数，借助名为FunctionExecutor的执行器。

首先，我们声明我们的函数参数：

```java
public class Weather {
    @JsonPropertyDescription("City and state, for example: León, Guanajuato")
    public String location;
    @JsonPropertyDescription("The temperature unit, can be 'celsius' or 'fahrenheit'")
    @JsonProperty(required = true)
    public WeatherUnit unit;
}
public enum WeatherUnit {
    CELSIUS, FAHRENHEIT;
}
public static class WeatherResponse {
    public String location;
    public WeatherUnit unit;
    public int temperature;
    public String description;
    
    // constructor
}
```

接下来，我们声明函数本身并将其与执行器关联，在本示例中，我们将从某个API中伪造一个响应：

```java
ChatFunction.builder()
        .name("get_weather")
        .description("Get the current weather of a location")
        .executor(Weather.class, w -> new WeatherResponse(w.location, w.unit, new Random().nextInt(50), "sunny"))
        .build()
```

然后，我们使用'service'模块中的FunctionExecutor对象来协助执行和转换为准备好进行对话的对象：

```java
List<ChatFunction> functionList = // list with functions
FunctionExecutor functionExecutor = new FunctionExecutor(functionList);

List<ChatMessage> messages = new ArrayList<>();
ChatMessage userMessage = new ChatMessage(ChatMessageRole.USER.value(), "Tell me the weather in Barcelona.");
messages.add(userMessage);
ChatCompletionRequest chatCompletionRequest = ChatCompletionRequest
        .builder()
        .model("gpt-3.5-turbo-0613")
        .messages(messages)
        .functions(functionExecutor.getFunctions())
        .functionCall(new ChatCompletionRequestFunctionCall("auto"))
        .maxTokens(256)
        .build();

ChatMessage response

Message = service.createChatCompletion(chatCompletionRequest).getChoices().get(0).getMessage();
ChatFunctionCall functionCall = responseMessage.getFunctionCall(); // might be null, but in this case it is certainly a call to our 'get_weather' function.

ChatMessage functionResponseMessage = functionExecutor.executeAndConvertToMessageHandlingExceptions(functionCall);
messages.add(response);
```

> **注意：** `FunctionExecutor`类是'service'模块的一部分。

您还可以创建自己的函数执行器。`ChatFunctionCall.getArguments()`的返回对象是JsonNode，为了简便起见，应该能够帮助您完成此操作。

要了解更详细的信息，请参阅在[OpenAiApiFunctionsExample.java](example/src/main/java/example/OpenAiApiFunctionsExample.java)中使用函数的对话示例。或者在使用函数和流的示例中：[OpenAiApiFunctionsWithStreamExample.java](example/src/main/java/example/OpenAiApiFunctionsWithStreamExample.java)

### 流程线程关闭

如果要在流式响应之后立即关闭进程，请调用`OpenAiService.shutdownExecutor()`。  
对于非流式调用，这是不必要的。

## 运行示例项目

[示例](example/src/main/java/example/OpenAiApiExample.java)项目所需的全部内容只是您的OpenAI API令牌

```bash
export OPENAI_TOKEN="sk-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
```

您可以使用以下命令尝试此项目的所有功能：

```bash
./gradlew runExampleOne
```

您还可以尝试使用新功能的能力：

```bash
./gradlew runExampleTwo
```

或者启用了 'stream' 模式的函数：

```bash
./gradlew runExampleThree
```

## 常见问题

### 是否支持GPT-4？

是的！GPT-4使用ChatCompletion API，您可以在[此处](https://platform.openai.com/docs/models/gpt-4)查看最新的模型选项。  
截至目前（截至4/1/23），GPT-4目前处于有限的测试版状态，请确保您有权访问再尝试使用。

### 是否支持函数？

绝对支持！非常容易使用自己的函数，而不必担心做脏活。  
如上所述，您可以参考[OpenAiApiFunctionsExample.java](example/src/main/java/example/OpenAiApiFunctionsExample.java)或 
[OpenAiApiFunctionsWithStreamExample.java](example/src/main/java/example/OpenAiApiFunctionsWithStreamExample.java)项目进行示例。 

### 为什么会出现连接超时？

请确保OpenAI在您所在的国家/地区可用。

### 为什么OpenAiService不支持x配置选项？

许多项目使用OpenAiService，为了最好地支持它们，我将其保持得非常简单。  
您可以创建自己的OpenAiApi实例来自定义头部、超时、基础URL等。  
如果您想要像重试逻辑和异步调用之类的功能，您将不得不创建一个 `OpenAiApi` 实例并直接调用它，而不是使用 `OpenAiService`。

## 废弃的端点

OpenAI已经弃用了基于引擎的端点，转而使用基于模型的端点。  
例如，不要使用 `v1/engines/{engine_id}/completions`，改为使用 `v1/completions` 并在 `CompletionRequest` 中指定模型。  
代码中包含所有废弃端点的升级说明。

在OpenAI关闭它们之前，我不会从此库中删除旧的端点。

## 许可证

根据MIT许可证发布
