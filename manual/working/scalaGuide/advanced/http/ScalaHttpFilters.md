<!--- Copyright (C) 2009-2015 Typesafe Inc. <http://www.typesafe.com> -->
<!--
# Filters
-->
# フィルター

<!--
Play provides a simple filter API for applying global filters to each request.
-->
Play は、あらゆるリクエストに適用するグローバルフィルター向けの、シンプルなフィルター API を提供しています。

<!--
## Filters vs action composition
-->
## フィルター vs アクション合成

<!--
The filter API is intended for cross cutting concerns that are applied indiscriminately to all routes.  For example, here are some common use cases for filters:
-->
フィルター API は、すべてのルートに無差別に適用される横断的な関心事を対象としています。フィルターの一般的なユースケースは、例えば以下のようなものです。

<!--
* Logging/metrics collection
* [[GZIP encoding|GzipEncoding]]
* [[Security headers|SecurityHeaders]]
-->
* ロギング / メトリクス コレクション
* [[GZIP エンコーディング|GzipEncoding]]
* [[セキュリティヘッダ|SecurityHeaders]]

<!--
In contrast, [[action composition|ScalaActionsComposition]] is intended for route specific concerns, such as authentication and authorisation, caching and so on.  If your filter is not one that you want applied to every route, consider using action composition instead, it is far more powerful.  And don't forget that you can create your own action builders that compose your own custom defined sets of actions to each route, to minimise boilerplate.
-->
対照的に、[[アクション合成|ScalaActionsComposition]] は認証や認可、キャッシュなど、特定のルートに対する関心事を対象としています。もし、フィルターをすべてのルートに適用したいのでなければ、代わりにアクション合成の使用を検討してみてください。アクション合成はフィルターよりも遙かに強力です。また、定型的なコードを最小限にするために、ルート毎に独自の定義済みアクション群を構成する、アクションビルダーを作成できることも忘れないでください。

<!--
## A simple logging filter
-->
## シンプルなロギングフィルター

<!--
The following is a simple filter that times and logs how long a request takes to execute in Play framework:
-->
以下は、Play framework があるリクエストを処理するためにどれくらい時間が掛かったのか計測してロギングする、シンプルなフィルターです。

@[simple-filter](code/ScalaHttpFilters.scala)

<!--
Let's understand what's happening here.  The first thing to notice is the signature of the `apply` method.  It's a curried function, with the first parameter, `nextFilter`, being a function that takes a request header and produces a result, and the second parameter, `requestHeader`, being the actual request header of the incoming request.
-->
ここで何が起こっているのかを見てみましょう。最初に注目すべきは `apply` メソッドのシグネチャです。これはカリー化された関数です。最初のパラメータ `nextFilter` はリクエストヘッダを取得して結果を生成する関数で、次のパラメータ `requestHeader` は受信するリクエストの実際のリクエストヘッダです。

<!--
The `nextFilter` parameter represents the next action in the filter chain. Invoking it will cause the action to be invoked.  In most cases you will probably want to invoke this at some point in your future.  You may decide to not invoke it if for some reason you want to block the request.
-->
`nextFilter` パラメータはフィルタチェーン内の次のアクションを表します。フィルタチェーンを呼び出すことで、次のアクションが呼び出されます。ほとんどの場合、未来のある同じ時点でこのフィルタを起動したい場合が多くあるでしょう。なんらかの理由でリクエストをブロックしたい場合は、これを呼び出したくない場合があるかもしれません。

<!--
We save a timestamp before invoking the next filter in the chain. Invoking the next filter returns a `Future[Result]` that will redeemed eventually. Take a look at the [[Handling asynchronous results|ScalaAsync]] chapter for more details on asynchronous results. We then manipulate the `Result` in the `Future` by calling the `map` method with a closure that takes a `Result`. We calculate the time it took for the request, log it and send it back to the client in the response headers by calling `result.withHeaders("Request-Time" -> requestTime.toString)`.
-->
チェーン内で次のフィルタを呼び出す前にタイムスタンプを保持します。次のフィルタを呼び出すと、最終的に使用される `Future[Result]` が返されます。非同期レスポンスの詳細については、[[非同期レスポンスの処理|ScalaAsync]] の章を見てください。`Result` を受け取るクロージャを使って `map` メソッドを呼び出すことによって、`Future` 内の `Result` を操作します。リクエストに要した時間を計算し、ログに記録し、`result.withHeaders("Request-Time" -> requestTime.toString)` を呼び出してレスポンスヘッダでクライアントに送信します。

<!--
## Using filters
-->
## フィルターを使う

<!--
The simplest way to use a filter is to provide an implementation of the [`HttpFilters`](api/scala/play/api/http/HttpFilters.html) trait in the root package:
-->
もっともシンプルにフィルターを使うには、ルートパッケージで [`HttpFilters`](api/scala/play/api/http/HttpFilters.html) トレイトの実装を用意します。

@[filters](code/ScalaHttpFilters.scala)

<!--
If you want to have different filters in different environments, or would prefer not putting this class in the root package, you can configure where Play should find the class by setting `play.http.filters` in `application.conf` to the fully qualified class name of the class.  For example:
-->
異なる環境で異なるフィルターを使いたい場合や、このクラスをルートパッケージに入れたくない場合は、`application.conf` の中の `play.http.filters` にクラスの完全修飾クラス名を設定することで、Play がクラスを見つけるべき場所を設定できます。例を示します。

    play.http.filters=com.example.MyFilters

<!--
## Where do filters fit in?
-->
## フィルターはどこに合う?

<!--
Filters wrap the action after the action has been looked up by the router.  This means you cannot use a filter to transform a path, method or query parameter to impact the router.  However you can direct the request to a different action by invoking that action directly from the filter, though be aware that this will bypass the rest of the filter chain.  If you do need to modify the request before the router is invoked, a better way to do this would be to place your logic in `Global.onRouteRequest` instead.
-->
フィルターは、アクションがルーターによって見つけられた後に、そのアクションをラップします。これは、フィルターを使ってルーターに影響を与えるパスやメソッド、そしてクエリパラメーターを変換できないことを意味します。フィルターから別のアクションを実行することで、リクエストをそのアクションに移動させてしまうこともできますが、これをするとフィルターチェーンの残りをバイパスしてしまうことに気を付けてください。ルーターが実行される前にリクエストを変更する必要がある場合は、フィルターを使う代わりに、そのロジックを `Global.onRouteRequest` に配置するのが、より良いやり方でしょう。

<!--
Since filters are applied after routing is done, it is possible to access routing information from the request, via the `tags` map on the `RequestHeader`.  For example, you might want to log the time against the action method.  In that case, you might update the `logTime` method to look like this:
-->
フィルターはルーティングが完了した後に適用されるので、`RequestHeader` の `tags` マップによってリクエストからルーティング情報にアクセスすることができます。例えば、アクションメソッドに対する実行時間をログに出力したいとします。この場合、`logTime` を以下のように書き換えることができます。

@[routing-info-access](code/FiltersRouting.scala)

<!--
> Routing tags are a feature of the Play router.  If you use a custom router, or return a custom action in `Global.onRouteRequest`, these parameters may not be available.
-->
> ルーティングタグは、Play ルーターの機能です。カスタムルーターを使うか、`Global.onRouteRequest` でカスタムアクションを返すと、これらのパラメータは利用できないかもしれません。

<!--
## More powerful filters
-->
## より強力なフィルター

<!--
Play provides a lower level filter API called `EssentialFilter` which gives you full access to the body of the request.  This API allows you to wrap [[EssentialAction|HttpApi]] with another action.
-->
Play は リクエストボディ全体にアクセスすることのできる、`EssentialFilter` と呼ばれるより低レベルなフィルター API を提供しています。この API により、[[EssentialAction|HttpApi]] を他のアクションでラップすることができます。

<!--
Here is the above filter example rewritten as an `EssentialFilter`:
-->
上記のフィルター例を `EssentialFilter` として書き直すと、以下のようになります。

@[essential-filter-example](code/EssentialFilter.scala)

<!--
The key difference here, apart from creating a new `EssentialAction` to wrap the passed in `next` action, is when we invoke next, we get back an `Iteratee`.  You could wrap this in an `Enumeratee` to do some transformations if you wished.  We then `map` the result of the iteratee and thus handle it.
-->
`next` アクションとして渡されたものをラップするために `EssentialAction` を新しく作成していることは別として、ここでの主な違いは、`next` アクションを呼び出したときに `Iteratee` を受け取ることです。これを `Enumeratee` でラップしていくつかの変換を行うこともできます。その後、iteratee の結果を `map` して処理します。

<!--
> Although it may seem that there are two different filter APIs, there is only one, `EssentialFilter`.  The simpler `Filter` API in the earlier examples extends `EssentialFilter`, and implements it by creating a new `EssentialAction`.  The passed in callback makes it appear to skip the body parsing by creating a promise for the `Result`, while the body parsing and the rest of the action are executed asynchronously.
-->
> 2 つの異なるフィルタ API があるように見えるかもしれませんが、ただ 1 つ `EssentialFilter` があるだけです。以前の例の、より簡単な `Filter` API は、`EssentialFilter` を継承し、新しい `EssentialAction` を作成することでそれを実装します。コールバックに渡された次のアクションは、ボディ解析と残りのアクションが非同期に実行されている間、`Result` の promise を作ることでボディ解析をスキップするように見せます。
