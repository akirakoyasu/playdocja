<!-- translated -->
<!--
# Streaming HTTP responses
-->
# HTTPレスポンスのストリーミング

<!--
## Standard responses and Content-Length header
-->
## 標準的なレスポンスとContent-lengthヘッダ

<!--
Since HTTP 1.1, to keep a single connection open to serve several HTTP requests and responses, the server must send the appropriate `Content-Length` HTTP header along with the response.

By default, when you send a simple result, such as:
-->
HTTP 1.1以降、複数のHTTPリクエストとレスポンスにまたがって単一のコネクションを使いまわすためには、サーバはレスポンスと一緒に適切な`Content-Length` HTTPヘッダを送信する必要があります。

デフォルトでは、例えば次のような単純な結果を送る場合を考えてみましょう。

```
public static Result index() {
  return ok("Hello World")
}
```

<!--
You are not specifying a `Content-Length` header. Of course, because the content you are sending is well known, Play is able to compute the content size for you and to generate the appropriate header.

> **Note** that for text-based content this is not as simple as it looks, since the `Content-Length` header must be computed according the encoding used to translate characters to bytes.

To be able to compute the `Content-Length` header properly, Play must consume the whole response data and load its content into memory.
-->
この例では、`Content-Length`を指定していません。送信しようとしているコンテンツが明らかなので、Playが自動的にコンテンツサイズを計算して、適切なヘッダを生成することができます。

> このようなテキストベースのコンテンツを返すことは見た目ほど単純なことではありません。というのは、文字列をバイト列に変換するために使われたエンコーディングにしたがって`Content-Length`をヘッダを計算しなければならないからです。

`Content-Length`ヘッダを正しく計算するため、Playはレスポンスのデータ全てを読んで、内容をメモリにロードします。

<!--
## Serving files
-->
## ファイルを送信する

<!--
If it’s not a problem to load the whole content into memory for simple content what about a large data set? Let’s say we want to send back a large file to the web client.

Play provides easy to use helpers to this common task of serving a local file:
-->
シンプルなコンテンツであればメモリに全て読み込んで処理しても問題ありませんが、データセットが巨大な場合はどうでしょうか？例えば、大きなファイルをWebクライアントへ送り返すような場合です。

Playには、ローカルファイルを送信するというよくあるタスクを簡単にするためのヘルパが用意されています。

```
public static Result index() {
  return ok(new java.io.File("/tmp/fileToServe.pdf"));
}
```

<!--
Additionally this helper will also compute the `Content-Type` header from the file name. And it will also add the `Content-Disposition` header to specify how the web browser should handle this response. The default is to ask the web browser to download this file by using `Content-Disposition: attachment; filename=fileToServe.pdf`.
-->
このヘルパはファイル名から`Content-Type`ヘッダを計算してくれます。さらに、Webブラウザがどのようにこのレスポンスを取り扱うべきかを指定する`Content-Disposition`ヘッダも追加してくれます。デフォルトでは、`Content-Disposition: attahment; filename=fileToServe.pdf`という指定により、Webブラウザはこのファイルのダウンロードをするかどうかをユーザに確認します。

<!--
## Chunked responses
-->
## チャンクレスポンス

<!--
For now, this works well with streaming file content, since we are able to compute the content length before streaming it. But what about dynamically-computed content with no content size available?

For this kind of response we have to use **Chunked transfer encoding**.
-->
今のところ、ストリーミングを始める前にコンテンツの長さを計算することができるため、うまくファイルの内容をストリーミングすることができていました。しかし、コンテンツのサイズが事前にわからないような動的に生成されるコンテンツをストリーミングする場合はどうでしょうか？

このような種類のレスポンスを返すためには、**チャンク転送エンコーディング**を利用します。

<!--
> **Chunked transfer encoding** is a data transfer mechanism in version HTTP 1.1 in which a web server serves content in a series of chunks. This uses the `Transfer-Encoding` HTTP response header instead of the `Content-Length` header, which the protocol would otherwise require. Because the `Content-Length` header is not used, the server does not need to know the length of the content before it starts transmitting a response to the client (usually a web browser). Web servers can begin transmitting responses with dynamically-generated content before knowing the total size of that content.
>
> The size of each chunk is sent right before the chunk itself so that a client can tell when it has finished receiving data for that chunk. The data transfer is terminated by a final chunk of length zero.
>
> [[http://en.wikipedia.org/wiki/Chunked_transfer_encoding]]
-->
> **チャンク転送エンコーディング**はHTTP 1.1で定義されているデータ転送メカニズムの一つで、Webサーバがコンテンツをいくつかのチャンクに分けて送信する、というものです。このレスポンスを送信するためには`Content-Length`ヘッダの代わりに`Transfer-Encoding`HTTPレスポンスヘッダを使います。`Content-Length`ヘッダがないので、サーバはレスポンスをクライアント（通常はWebブラウザ）へ送信し始める前にコンテンツの長さを知る必要はありません。つまり、Webサーバは動的に生成されるコンテンツの最終的な長さを知ることなく、レスポンスを送り始めることができます。
>
> 各チャンクの長さはチャンクの内容の直前に送信されます。これによって、クライアントはチャンクの受信が終わったことを認識できます。最後に、長さがゼロのチャンクを送信すると、データ転送は完了です。
> [[http://en.wikipedia.org/wiki/Chunked_transfer_encoding]]

<!--
The advantage is that we can serve data **live**, meaning that we send chunks of data as soon as they are available. The drawback is that since the web browser doesn’t know the content size, it is not able to display a proper download progress bar.

Let’s say that we have a service somewhere that provides a dynamic `InputStream` that computes some data. We can ask Play to stream this content directly using a chunked response:
-->
この方式の利点は、データを**ライブ**に提供できる、つまり利用できるようになったデータを即座にチャンクとして送信できることです。欠点は、Webブラウザがコンテンツサイズを知らないため、ダウンロードプログレスバーを正しく表示できないということです。

例として、あるサービスが何かのデータを計算して動的な`InputStream`を提供しているケースを考えます。チャンクレスポンスを利用して、Playにコンテンツを直接的にストリーミングさせるには、次のようにします。

```
public static Result index() {
  InputStream is = getDynamicStreamSomewhere();
  return ok(is);
}
```

<!--
You can also set up your own chunked response builder. The Play Java API supports both text and binary chunked streams (via `String` and `byte[]`):  
-->
自分でチャンクレスポンスのビルダをつくることもできます。Play Java APIはテキスト形式とバイナリ（それぞれ`String`と`byte[]`経由）、両方のチャンクストリームに対応しています。

```
public static index() {
  // Prepare a chunked text stream
  Chunks<String> chunks = new StringChunks() {
   
    // Called when the stream is ready
    public void onReady(Chunks.Out<String> out) {
      registerOutChannelSomewhere(out);
    }
   
  }
 
  // Serves this stream with 200 OK
  ok(chunks);
}
```

<!--
The `onReady` method is called when it is safe to write to this stream. It gives you a `Chunks.Out` channel you can write to.

Let’s say we have an asynchronous process (like an `Actor`) somewhere pushing to this stream:
-->
`onReady`メソッドはストリームへの書き込み準備が整ったときに呼び出されます。引数には書き込み先のチャンネルである`Chunks.Out`が渡されます。

さらに、（`アクター`のような）非同期プロセスがこのストリームにデータを書き込むとします。

```
public void registerOutChannelSomewhere(Chunks.Out<String> out) {
  out.write("kiki");
  out.write("foo");
  out.write("bar");
  out.close();
}
```

<!--
We can inspect the HTTP response sent by the server:  
-->
サーバから送信されるHTTPレスポンスは次のようになります。

```
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked

4
kiki
3
foo
3
bar
0

```

<!--
We get three chunks and one final empty chunk that closes the response.  
-->
３つのチャンクと、最後にレスポンスを完了するための空のチャンクが送信されていますね。

<!--
> **Next:** [[Comet sockets | JavaComet]]
-->
> **次ページ:** [[Cometソケット | JavaComet]]
