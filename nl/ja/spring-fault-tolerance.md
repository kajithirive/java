---

copyright:
  years: 2018, 2019
lastupdated: "2019-04-22"

keywords: fault tolerance spring, hystrix spring, netflix spring, hystrixcommand spring, bulkhead spring, circuit breaker spring

subcollection: java

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}
{:tip: .tip}
{:note: .note}

# Spring によるフォールト・トレランス
{: #spring-tolerance}

回復力のあるシステムを作成すると、その中のすべてのサービスに要件が課されます。 クラウド環境には動的な性質があるため、予想外の事態に適切に対応するようにサービスを設計することが必要です。

Spring では [Netflix Hystrix](https://github.com/Netflix/Hystrix/wiki){: new_window} ![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン") フォールト・トレランス・ライブラリーを使用して、アプリケーション・レベルのフォールト・トレランスの問題をサポートしています。 Hystrix を使用すると、フォールバック、回路ブレーカー、バルクヘッド、そして関連するメトリックを作成できます。

この情報は、[クラウド・ネイティブ開発: フォールト・トレランス](/docs/java?topic=cloud-native-fault-tolerance#fault-tolerance)で説明されているフォールト・トレランスの事例に基づいています。
{: note}

## Spring アプリケーションによる Hystrix の使用
{: #spring-hystrix-starter}

Spring アプリケーションで Hystrix を使いやすくするために、アプリケーション内で Hystrix を統合するアノテーションを使った方法を可能にする Spring Boot スターターがあります。

Hystrix を追加するには、以下の依存関係を追加します。

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```
{: codeblock}

この新しい関数を Spring で使用するには、メイン・アプリケーション・クラスにアノテーションを追加し、`@EnableCircuitBreaker` を追加して回路遮断用の Hystrix アノテーションの Spring 処理を有効にします。

```java
@SpringBootApplication
@EnableCircuitBreaker
public class MyApplication {

	public static void main(String[] args) {
		SpringApplication.run(MyApplication.class, args);
	}
```
{: codeblock}

### HystrixCommand を使用してフォールバックを定義する
{: #spring-fallback}

回路ブレーカー動作をメソッドに追加するには、メソッドに `@HystrixCommand` アノテーションを追加します。 Spring は、`@EnableCircuitBreaker` でアノテーションが付けられているアプリケーション内のメソッドを発見し、エラーとタイムアウトをモニターするための Hystrix サポートでそれらをラップします。 次に、Spring は、必要なときに適切な代替動作を呼び出します。

`@HystrixCommand` アノテーションは、`@Component` または `@Service` 内のメソッドでのみサポートされます。
{: note}

以下の `@HystrixCommand` アノテーションは、`service()` 呼び出しをラップして回路ブレーカー動作を指定します。 `service()` メソッドが失敗した、または回路がオープンの場合、プロキシーは `fallback()` メソッドを呼び出します。

```java
@Autowired
private MicroService myService;

@GetMapping("/endpoint")
public String value() throws Exception {
    return "Service returned: " + myService.service();
}

@Service
class MicroService {
    @HystrixCommand(fallbackMethod = "fallback")
    public String service() {
        // Do some processing. If an exception is thrown
        // the fallback method will be called
        return "Success";
    }

    public String fallback(Throwable e) {
        return "Fallback! Reason: " + e.getMessage() + "\n";
    }
}
```
{: codeblock}

詳しくは、Spring ベースの [Hystrix 回路ブレーカー](https://spring.io/guides/gs/circuit-breaker/){: new_window} ![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン") の例を参照してください。
{: tip}

### タイムアウトの使用
{: #spring-timeout}

アプリケーションは、応答していないリモート・サービスに対してどのように応答しますか? しばらく待機するか、非常に時間がかかる可能性があります。 Hystrix には、アプリケーションに許容できる待機時間をユーザーが定義できる機能があります。

単純に `HystrixCommand` アノテーションに追加するだけで、タイムアウト値をデフォルト値の 1 秒から 30 秒の範囲で変更できます。

```java
@HystrixCommand(
    fallbackMethod = "fallback",
    commandProperties = {
        @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "30000"),
    }
)
```
{: codeblock}

### バルクヘッドの使用
{: #spring-bulkhead}

Hystrix はセマフォー・ベースとキュー・ベースの両方のバルクヘッドをサポートしています。 以下のスニペットは、4 つのスレッドを割り振り、未処理の要求を 10 に制限するキュー・ベースのバルクヘッドを構成する方法を示しています。

```java
@HystrixCommand(
    fallbackMethod = "fallback",
    threadPoolProperties = {
        @HystrixProperty(name = "coreSize", value = "4"),
        @HystrixProperty(name = "maxQueueSize", value = "10")
    }
)
```
{: codeblock}

### 回路ブレーカーのステータス
{: #spring-breaker-status}

Hystrix Spring スターターには特別の切り札があります。アプリケーションのデフォルトの `/health` エンドポイントを拡張できるのです。この機能は Spring Actuator を介して提供されます。 詳しくは、[Spring によるメトリック](/docs/java?topic=java-spring-metrics#spring-metrics)を参照してください。

ヘルス・エンドポイントが[追加の詳細情報を含めて構成されている](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html#production-ready-health){: new_window}![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン")場合、回路ブレーカーのステータスはヘルス・チェック情報とともに含まれます。 (この振る舞いはデフォルトでは無効になっています。)

```
{
    "hystrix": {
        "openCircuitBreakers": [
            "MicroService::service"
        ],
        "status": "CIRCUIT_OPEN"
    },
    "status": "UP"
}
```
{: screen}

## 次のステップ
{: #spring-tolerance-next-steps notoc}

Hystrix の構成について詳しくは、「[Hystrix Configuration Wiki](https://github.com/Netflix/Hystrix/wiki/Configuration){: new_window}」![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン")を参照してください。

Hystrix と Spring について詳しくは、以下を参照してください。

* [A Guide to Spring Cloud Netflix](https://www.baeldung.com/spring-cloud-netflix-hystrix){: new_window} ![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン")
* [Spring Circuit Breaker Guide](https://spring.io/guides/gs/circuit-breaker/){: new_window} ![外部リンク・アイコン](../icons/launch-glyph.svg "外部リンク・アイコン")
