# PHP 通过 OpenTelemetry 协议上报demo

#### 创建应用
1. 初始化应用

   ``` bash
   mkdir <project-name> && cd <project-name>
   
   
   composer init \
    --no-interaction \
    --stability beta \
    --require slim/slim:"^4" \
    --require slim/psr7:"^1"
   composer update
   ```
2. 编写业务代码


   在<project-name>目录下创建一个`index.php`文件，添加如下内容。


   本文档将使用一个扔骰子游戏（返回1-6之间的一个随机数）模拟业务代码。

   ``` php
   <?php
   use Psr\Http\Message\ResponseInterface as Response;
   use Psr\Http\Message\ServerRequestInterface as Request;
   use Slim\Factory\AppFactory;
   
   require __DIR__ . '/vendor/autoload.php';
   
   $app = AppFactory::create();
   
   $app->get('/rolldice', function (Request $request, Response $response) {
    $result = random_int(1,6);
    $response->getBody()->write(strval($result));
    return $response;
   });
   
   $app->run();
   ```

   此时应用已经编写完成，执行`php -S localhost:8080`命令即可运行应用，访问地址为`http://localhost:8080/rolldice`。


#### 

#### 构建OpenTelemetry PHP扩展

> **说明：**
> 
> 如果已经构建过OpenTelemetry PHP扩展，可跳过当前步骤。
> 

1. 下载构建OpenTelemetry PHP extension所需要的工具：

- macOS

   ``` bash
   brew install gcc make autoconf
   ```
- Linux（apt）

   ``` bash
   sudo apt-get install gcc make autoconf
   ```
2. 使用PECL构建OpenTelemetry PHP扩展：

   ``` bash
   pecl install opentelemetry-1.0.0beta6
   ```

   注意，构建成功时输出内容的最后几行如下（路径可能不完全一致）：

   ``` bash
   Build process completed successfully
   Installing '/opt/homebrew/Cellar/php/8.2.8/pecl/2020829/opentelemetry.so'
   install ok: channel://pecl.php.net/opentelemetry-1.0.0beta6
   Extension opentelemetry enabled in php.ini
   ```
3. 启用OpenTelemetry PHP扩展。
   

   > **说明：**
   > 
   > 如果上一步输出了`Extension opentelemetry enabled in php.ini`，表明已经启用，请跳过当前步骤。
   > 


   在`php.ini`文件中添加如下内容：

   ``` php
   [opentelemetry]
   extension=opentelemetry.so
   ```




4. 验证是否构建并启用成功
   ``` bash
   php -m | grep opentelemetry
   ```

   预期输出：

   ``` bash
   opentelemetry
   ```

#### 

#### 导入OpenTelemetry PHP SDK以及OpenTelemetry gRPC Explorer所需依赖
1. 下载PHP HTTP客户端库，用于链路数据上报。

   ``` bash
   composer require guzzlehttp/guzzle
   ```
2. 下载OpenTelemetry PHP SDK。

   ``` bash
   composer require \
    open-telemetry/sdk \
    open-telemetry/exporter-otlp
   ```
3. 下载使用gRPC上报数据时所需依赖。

   ``` bash
   pecl install grpc # 如果之前已经下载过grpc，可以跳过这一步
   composer require open-telemetry/transport-grpc
   ```



#### 编写OpenTelemetry初始化工具类
1. 在`index.php`文件所在目录中创建`opentelemetry_util.php`文件。

2. 在文件中添加如下代码：

   ``` php
   <?php包含设置应用名、Trace导出方式、Trace上报接入点，并创建全局TraceProvide
   
   use OpenTelemetry\API\Globals;
   use OpenTelemetry\API\Trace\Propagation\TraceContextPropagator;
   use OpenTelemetry\Contrib\Otlp\SpanExporter;
   use OpenTelemetry\SDK\Common\Attribute\Attributes;
   use OpenTelemetry\SDK\Common\Export\Stream\StreamTransportFactory;
   use OpenTelemetry\SDK\Resource\ResourceInfo;
   use OpenTelemetry\SDK\Resource\ResourceInfoFactory;
   use OpenTelemetry\SDK\Sdk;
   use OpenTelemetry\SDK\Trace\Sampler\AlwaysOnSampler;
   use OpenTelemetry\SDK\Trace\Sampler\ParentBased;
   use OpenTelemetry\SDK\Trace\SpanProcessor\SimpleSpanProcessor;
   use OpenTelemetry\SDK\Trace\SpanProcessor\BatchSpanProcessorBuilder;
   use OpenTelemetry\SDK\Trace\TracerProvider;
   use OpenTelemetry\SemConv\ResourceAttributes;
   use OpenTelemetry\Contrib\Grpc\GrpcTransportFactory;
   use OpenTelemetry\Contrib\Otlp\OtlpUtil;
   use OpenTelemetry\API\Signals;
   
   // OpenTelemetry 初始化配置（需要在PHP应用初始化时就进行OpenTelemetry初始化配置）
   function initOpenTelemetry()
   { 
    // 1. 设置 OpenTelemetry 资源信息
    $resource = ResourceInfoFactory::emptyResource()->merge(ResourceInfo::create(Attributes::create([
      ResourceAttributes::SERVICE_NAME => '<your-service-name>', # 应用名，必填，如php-manual-demo
      ResourceAttributes::HOST_NAME => '<your-host-name>' # 主机名，选填
      'token' => '<Token>' # 替换成步骤1中获得的Token信息
    ])));
   
   
    // 2. 创建将 Span 输出到控制台的 SpanExplorer
    // $spanExporter = new SpanExporter(
    // (new StreamTransportFactory())->create('php://stdout', 'application/json')
    // );
   
    // 2. 创建通过 gRPC 上报 Span 的 SpanExplorer
    $transport = (new GrpcTransportFactory())->create('<接入点>' . OtlpUtil::method(Signals::TRACE)); # 替换成步骤1中获得的接入点信息
    $spanExporter = new SpanExporter($transport);
   
   
    // 3. 创建全局的 TraceProvider，用于创建 tracer
    $tracerProvider = TracerProvider::builder()
    ->addSpanProcessor(
    (new BatchSpanProcessorBuilder($spanExporter))->build()
    )
    ->setResource($resource)
    ->setSampler(new ParentBased(new AlwaysOnSampler()))
    ->build();
   
    Sdk::builder()
    ->setTracerProvider($tracerProvider)
    ->setPropagator(TraceContextPropagator::getInstance())
    ->setAutoShutdown(true) // PHP 程序退出后自动关闭 tracerProvider，保证链路数据都被上报
    ->buildAndRegisterGlobal(); // 将 tracerProvider 添加到全局
   
   }
   ?>
   ```   
#### 修改应用代码，使用OpenTelemetry API创建Span
1. 在`index.php`文件中导入所需包：

   ``` php
   <?php
   
   use OpenTelemetry\API\Globals;
   use OpenTelemetry\SDK\Common\Attribute\Attributes;
   use OpenTelemetry\SDK\Trace\TracerProvider;
   
   require __DIR__ . '/opentelemetry_util.php';
   ```
2. 调用`initOpenTelemetry`方法完成初始化，需要在PHP应用初始化时就进行OpenTelemetry初始化配置：

   ``` php
   // OpenTelemetry 初始化，包含设置应用名、Trace导出方式、Trace上报接入点，并创建全局TraceProvider
   initOpenTelemetry();
   ```
3. 在`rolldice`接口中创建Span。

   ``` php
   /**
    * 1. 接口功能：模拟扔骰子，返回一个1-6之间的随机正整数
    * 并演示如何创建Span、设置属性、事件、带有属性的事件
    */
   $app->get('/rolldice', function (Request $request, Response $response) {
    // 获取 tracer
    $tracer = \OpenTelemetry\API\Globals::tracerProvider()->getTracer('my-tracer');
    // 创建 Span
    $span = $tracer->spanBuilder("/rolldice")->startSpan();
    // 为 Span 设置属性
    $span->setAttribute("http.method", "GET");
    // 为 Span 设置事件
    $span->addEvent("Init");
    // 设置带有属性的事件
    $eventAttributes = Attributes::create([
    "key1" => "value",
    "key2" => 3.14159,
    ]);
   
    // 业务代码
    $result = random_int(1,6);
    $response->getBody()->write(strval($result));
   
    $span->addEvent("End");
    // 销毁 Span
    $span->end();
   
    return $response;
   });
   ```
4. 创建嵌套Span。


   新建一个`rolltwodices`接口，模拟扔两个骰子，返回两个1-6之间的随机正整数。


   以下代码演示如何创建嵌套的Span：

   ``` php
   $app->get('/rolltwodices', function (Request $request, Response $response) {
    // 获取 tracer
    $tracer = \OpenTelemetry\API\Globals::tracerProvider()->getTracer('my-tracer');
    // 创建 Span
    $parentSpan = $tracer->spanBuilder("/rolltwodices/parent")->startSpan();
    $scope = $parentSpan->activate();
   
    $value1 = random_int(1,6);
   
    $childSpan = $tracer->spanBuilder("/rolltwodices/parent/child")->startSpan();
    
    // 业务代码
    $value2 = random_int(1,6);
    $result = "dice1: " . $value1 . ", dice2: " . $value2; 
   
    // 销毁 Span
    $childSpan->end();
    $parentSpan->end();
    $scope->detach();
   
    $response->getBody()->write(strval($result));
    return $response;
   });
   ```
5. 使用Span记录代码中发生的异常。


   新建`error`接口，模拟接口发生异常。


   以下代码演示如何在代码发生异常时使用Span记录状态：

   ``` php
   $app->get('/error', function (Request $request, Response $response) {
    // 获取 tracer
    $tracer = \OpenTelemetry\API\Globals::tracerProvider()->getTracer('my-tracer');
    // 创建 Span
    $span3 = $tracer->spanBuilder("/error")->startSpan();
    try {
    // 模拟代码发生异常
    throw new \Exception('exception!');
    } catch (\Throwable $t) {
    // 设置Span状态为error
    $span3->setStatus(\OpenTelemetry\API\Trace\StatusCode::STATUS_ERROR, "expcetion in span3!");
    // 记录异常栈轨迹
    $span3->recordException($t, ['exception.escaped' => true]);
    } finally {
    $span3->end();
    $response->getBody()->write("error");
    return $response;
    }
   });
   ```

#### 
### 完整接入文档可参考 [腾讯云可监控平台官方文档](https://cloud.tencent.com/document/product/248/101074)。



