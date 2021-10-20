---
layout: docs-ja
title: Cache
category: Manual
permalink: /manuals/1.0/ja/cache.html
---

# Cache

> There are only two hard things in Computer Science: cache invalidation and naming things.
>
> -- Phil Karlton

## 概要

BEAR.Sundayは従来のものと別次元のキャッシュ機能をサポートします。

* イベント駆動のキャッシュ無効化
* コンテンツの依存解決
* ドーナッツキャッシュ
* CDNコントロール
* 条件付きリクエスト

優れたキャッシュシステムはユーザー体験の本質的な質を向上させ、資源利用コストと環境負荷を下げます。

BEAR.Sundayのキャッシュシステムは動的コンテンツを可能な限り静的に扱えるよう設計され、計算資源とネットワークの利用の最小化に最善を尽くします。

## 分散キャッシュ

<img src="https://user-images.githubusercontent.com/529021/137062427-c733c832-0631-4a43-a6ee-4204e6be007c.png" alt="distributed cache">

REST制約に従った分散キャッシュシステムは、CPUだけでなくネットワーク資源も節約します。

PHPが直接扱うRedisやAPCなどの**サーバーサイドキャッシュ**、コンテンツ配信ネットワーク(CDN)として知られる**共有キャッシュ**、WebブラウザやAPIクライアントでキャッシュされる**クライアントサイドキャッシュ**、BEAR.SundayはこれらのキャッシュとモダンCDNを統合したキャッシングフレームワークを提供します。


## タグベースでのキャッシュ無効化

<img width="369" alt="dependency graph 2021-10-19 21 38 02" src="https://user-images.githubusercontent.com/529021/137910748-b6e95839-eeb7-4ade-a564-3cdcd5fdc09e.png">

コンテンツキャッシュには依存性の問題があります。コンテンツAがコンテンツBに依存し、BがCに依存している場合、Cが更新されるとCのキャッシュとETagだけでなく、Cに依存するBのキャッシュとETag、Bに依存するAのキャッシュとETagも更新されなければなりません。

BEAR.Sundayはそれぞれのリソースが依存リソースのURIをタグとして保持する事でこの問題を解決します。`#[Embed]`で埋め込まれたリソースに変更があると、関係する全てのリソースのキャッシュとETagが無効化され、次のリクエストのためにキャッシュの再生性が行われます。

## ドーナッツキャッシュ

<img width="200" alt="donut caching" src="https://user-images.githubusercontent.com/529021/137097856-f9428918-5b76-4c0e-8cea-2472c15d82e9.png">

ドーナツキャッシュはキャッシュの最適化のための**部分キャッシュ**技術の１つです。

あとで追記されるドーナツの穴を除いて、ドーナツ（ページ全体）をキャッシュします。例えばコメントを含むブログの記事ページではコメントが穴です。それ以外のビュー化された記事のドーナッツ部分をキャッシュして、穴部分のコメントを別途取得して埋め込みます。

穴部分の再計算が必要になった時にキャッシュされたドーナッツ部分の計算は再利用され、計算資源を節約する事ができます。

### リカーシブ・ドーナッツ

<img width="191" alt="recursive donut 2021-10-19 21 27 06" src="https://user-images.githubusercontent.com/529021/137909083-2c5176f7-edb7-422b-bccc-1db90460fc15.png">

ドーナッツ構造は再起され適用されます。
例えば、AがBを含み、BがCを含むコンテンツの場合に、Cが変更された時に、変更されたCの部分を除いてAのキャッシュとBのキャッシュは再利用されます。AやBのコンテンツ取得のためのDBアクセスやビューのレンダリングは行われませんが、AとBのキャッシュ、ETagは再生成されます。

最適化された部分キャッシュが、最小のコストでコンテンツ再生成を行います。

## イベントドリブン型コンテンツ

従来、CDNはアプリケーションロジックを必要とするコンテンツは「動的」であり、したがってCDNではキャッシュはできないと考えられてきました。FastlyやAkamaiなどの一部のCDNは即時または数秒以内でのタグベースでのキャッシュ無効化が可能になり、[この考えは過去のもの](https://www.fastly.com/blog/leveraging-your-cdn-cache-uncacheable-content)になろうとしています。

BEAR.Sundayの依存解決はサーバーサイドだけでなく共有キャッシュでも行われます。AOPが変更を検知し共有キャッシュにPURGEリクエストを行うことで、サーバーサイドと同じように共有キャッシュ上の関連キャッシュの無効化が行われます。

## 条件付きリクエスト

<img width="468" alt="conditional request" src="https://user-images.githubusercontent.com/529021/137151061-8d7a5605-3aa3-494c-91c5-c1deddd987dd.png">

コンテンツの変更はAOPで管理され、コンテンツのエンティティタグ(ETag)は自動で更新されます。ETagを使ったHTTPの条件付きリクエストは計算資源の利用を最小化するだけでなく、`304 Not Modified`を返すだけの応答はネットワーク資源の利用も最小化します。


# 利用法

キャッシュ対象のクラスに`#[CacheableResponse]`とアトリビュートを付与します。

```php
use BEAR\RepositoryModule\Annotation\DonutCache;

#[CacheableResponse]
class BlogPosting extends ResourceObject
{
    public $headers = [
        RequestHeader::CACHE_CONTROL => CacheControl::NO_CACHE
    ];

    #[Embed(rel: "comment", src: "page://self/html/comment")]
    public function onGet(int $id = 0): static
    {
        $this->body['article'] = 'hello world';

        return $this;
    }

    public function onDelete(int $id = 0): static
    {
        return $this;
    }
}
```

キャッシュ対象メソッドを選択したい場合はクラスに属性を指定しないで、メソッドに指定します。その場合はキャッシュ変更メソッドに`#[RefreshCache]`という属性を付与します。

```php
class Todo extends ResourceObject
{
    #[CacheableResponse]
    public function onPut(int $id = 0, string $todo): static
    {
    }

    #[RefreshCache]
    public function onDelete(int $id = 0): static
    {
    }	
}
```

どちらかの方法でアトリビュートを付与すると、概要で紹介した全ての機能が適用されます。
イベントドリブン型コンテンツを想定してデフォルトでは時間(TTL)によるキャッシュの無効化は行われません

## TTL

TTLの指定は`DonutRepositoryInterface::put()`で行います。
`ttl`はドーナツの穴以外のキャッシュ時間、`sMaxAge`はCDNのキャッシュ時間です。

```php
class BlogPosting extends ResourceObject
{
    public function __construct(private DonutRepositoryInterface $repository)
    {}

    #[Embed(rel: "comment", src: "page://self/html/comment")]
    public function onGet(): static
    {
        // ....
        $this->repository->put($this, ttl:10, sMaxAge:100);　

        return $this;
    }
}
```
### TTLの既定値

イベントドリブン型コンテンツでは、コンテンツが変更されたらキャッシュにすぐに反映されなければなりません。そのため、既定値のTTLはCDNのモジュールのインストールによって変わります。CDNがタグベースでのキャッシュ化を無効化をサポートしていればTTLは無期限（1年間）で、サポートの無い場合には10秒です。

つまりキャッシュ反映時間は、Fastlyなら即時、Akamaiなら数秒、それ以外なら10秒が期待される時間です。

カスタマイズするには`CdnCacheControlHeader`を参考に`CdnCacheControlHeaderSetterInterface`を実装して束縛します。

## キャッシュ無効化

手動でキャッシュを無効化するには`DonutRepositoryInterface`を利用します。
指定されたキャッシュだけでなく、そのETag、指定したリソースの依存にしている他のリソースのレスポンス（単数または複数）のキャッシュとそのETagがサーバーサイド、（可能な場合）CDN共に無効化されます。

```php
interface DonutRepositoryInterface
{
    public function purge(AbstractUri $uri): void;
    public function invalidateTags(array $tags): void;
}
```

### URIによる無効化

```php
// example
$this->repository->purge(new Uri('app://self/blog/comment'));
```

### タグによる無効化

```php
$this->repository->invalidateTags(['template_a', 'campaign_b']);
```
### CDNでタグの無効化

CDNでタグベースでのキャッシュ無効化を有効にするためには`PurgerInterface`を実装して束縛する必要があります。

```php
use BEAR\QueryRepository\PurgerInterface;

interface PurgerInterface
{
    public function __invoke(string $tag): void;
}
```

### 依存タグの指定

PURGE用のキーを指定するためには`SURROGATE_KEY`ヘッダーで指定します。複数文字列の時はスペースをセパレータに使います。

```php
use BEAR\QueryRepository\Header;

class Foo
{
    public $headers = [
        Header::SURROGATE_KEY => 'template_a campaign_b'
    ];
```

`template_a`または`campaign_b`のタグによるキャッシュの無効化が行われた場合、FooのキャッシュとFooのETagはサーバーサイド、CDN共に無効になります。

### リソースの依存

`UriTagInterface`を使ってURIを依存タグ文字列に変換します。

```php
public function __construct(private UriTagInterface $uriTag)
{}
```
```php
$this->headers[Header::SURROGATE_KEY] = ($this->uriTag)(new Uri('app://self/foo'));
```

`app://self/foo`に変更があった場合にこのキャッシュはサーバーサイド、CDN共に無効化されます。

### 連想配列をリソースの依存に

```php
// bodyの内容
[
    ['id' => '1', 'name' => 'a'],
    ['id' => '2', 'name' => 'b'],
]
```
上記のような`body`連想配列から、依存するURIタグリストを生成する場合は`fromAssoc()`メソッドでURIテンプレートを指定します。

```php
$this->headers[Header::SURROGATE_KEY] = $this->uriTag->fromAssoc(
    uriTemplate: 'app://self/item{?id}',
    assoc: $this->body
);
```

上記の場合、`app://self/item?id=1`及び`app://self/item?id=2`に変更があった場合に、このキャッシュはサーバーサイド、CDN共に無効化されます。

## CDN

特定CDN対応のモジュールをインストールするとベンダー固有のヘッダーが出力されます。

```php
$this->install(new FastlyModule())
$this->install(new AkamaiModule())
```

## マルチCDN

CDNを多段構成にして、役割に応じたTTLを設定することもできます。例えばこの図では上流に多機能なCDNを配置して、下流にはコンベンショナルなCDNを配置しています。コンテンツのインバリデーションなどは上流のCDNに対して行い、下流のCDNはそれを利用するようにします。

<img width="344" alt="multi cdn diagram" src="https://user-images.githubusercontent.com/529021/137098809-ec949a15-8efb-4d03-9808-3be15523ade7.png">


# レスポンスヘッダー

CDNのキャッシュコントロールについてはBEAR.Sundayが自動で行いCDN用のヘッダーを出力します。クライアントのキャッシュコントロールはコンテンツに応じてResourceObjectの`$header`に記述します。

セキュリティやメンテナンスの観点からこのセクションは重要です。
全てのResourceObjectで`Cache-Control`を指定するようにしましょう。

### キャッシュ不可

キャッシュができないコンテンツは必ず指定しましょう。

```php
ResponseHeader::CACHE_CONTROL => CacheControl::NO_STORE
```

### 条件付きリクエスト

サーバーにコンテンツ変更がないかを確認してから、キャッシュを利用します。サーバーサイドのコンテンツの変更は検知され反映されます。

```php
ResponseHeader::CACHE_CONTROL => CacheControl::NO_CACHE
```

### クライアントキャッシュ時間の指定

クライントでキャッシュされます。最も効率的なキャッシュですが、サーバーサイドでコンテンツが変更されても指定した時間に反映されません。

またブラウザのリロード動作ではこのキャッシュは利用されません。`<a>`タグで遷移、またはURL入力した場合にキャッシュが利用されます。

```php
ResponseHeader::CACHE_CONTROL => 'max-age=60'
```

レスポンス速度を重視する場合には、SWRの指定も検討しましょう。

```php
ResponseHeader::CACHE_CONTROL => 'max-age=30 stale-while-revalidate=10'
```

この場合、max-ageの30秒を超えた時にオリジンサーバーからフレッシュなレスポンス取得が完了するまで、SWRで指定された最大10秒間はそれまでの古いキャッシュ(stale)レスポンスを返します。つまりキャッシュが更新されるのは最後のキャッシュ更新から30秒から40秒間の間のいずれかになりますが、どのリクエストもキャッシュからの応答になり高速です。

#### RFC7234対応クライアント

APIでクライアントキャッシュを利用する場合にはRFC7234対応APIクライアントを利用します。

* iOS [NSURLCache](https://nshipster.com/nsurlcache/)
* Android [HttpResponseCache](https://developer.android.com/reference/android/net/http/HttpResponseCache)
* PHP [guzzle-cache-middleware](https://github.com/Kevinrob/guzzle-cache-middleware)
* JavaScript(Node) [cacheable-request](https://www.npmjs.com/package/cacheable-request)
* Go [lox/httpcache](https://github.com/lox/httpcache)
* Ruby [faraday-http-cache](https://github.com/plataformatec/faraday-http-cache)
* Python [requests-cache](https://pypi.org/project/requests-cache/)

### プライベート

キャッシュを他のクライアントと共有しない時には`private`を指定します。クライアントサイドのみキャッシュ保存されます。この場合サーバーサイドではキャッシュ指定をしないようにします。

```php
ResponseHeader::CACHE_CONTROL => 'private, max-age=30'
```

> 共用キャッシュを利用する場合でもほとんどの場合において`public`を指定する必要はありません。

## キャッシュ設計

API（またはコンテンツ）は**情報API** (Information API)、**計算API** (Computation API)は２つに分けることができます。計算APIは再現が難しく真に動的でキャッシュに不適なコンテンツです。一方の情報APIはDBから読み出され、PHPで加工されたとしても本質的には静的なコンテンツのAPIです。

適切なキャシュを適用するためにコンテンツを分析します。

* 情報APIか計算APIか
* 依存関係は
* 内包関係は
* 無効化はイベントがトリガーか、それともTTLか
* イベントはアプリケーションが検知可能か、監視が必要か
* TTLは予測可能か不可能か

キャッシュ設計をアプリケーション設計プロセスの一部にして、仕様にする事も検討しましょう。 ライフサイクルを通してプロジェクトの安全性にも寄与するはずです。

### アダプティブ TTL

コンテンツの生存期間が予測可能で、その期間にイベントによる更新が行われない時はそれをクライントやCDNに正しく伝えます。例えば株価のAPIを扱う時、現在が金曜日の夜だとすると月曜の取引開始時間までは情報更新が行われない事が分かっています。その時間までの秒数を計算してTTLとして指定し、取引時間の時には適切なTTLを指定します。

クライアントは更新がないと分かっているリソースにリクエストする必要はありません。

## 用語

* [条件付きリクエスト](https://developer.mozilla.org/ja/docs/Web/HTTP/Conditional_requests)
* [ETag (バージョン識別子)](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/ETag)
* [イベントドリブン型コンテンツ](https://www.fastly.com/blog/rise-event-driven-content-or-how-cache-more-edge)
* [ドーナッツキャッシュ / 部分キャッシュ](https://www.infoq.com/jp/news/2011/12/MvcDonutCaching/)
* [サロゲートキー / タグベースの無効化](https://docs.fastly.com/ja/guides/getting-started-with-surrogate-keys)
* ヘッダー
  * [Cache-Control](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Cache-Control)
  * [CDN-Cache-Control](https://blog.cloudflare.com/cdn-cache-control/)
  * [Vary](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Vary)
  * [Stale-While-Revalidate (SWR)](https://www.infoq.com/jp/news/2020/12/ux-stale-while-revalidate/)