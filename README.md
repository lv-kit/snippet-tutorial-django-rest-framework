# 第07回: スキーマとクライアントライブラリ  
スキーマというのはコンピュータ向けに最適化されたドキュメントのことで、利用可能なAPIエンドポイント、そのURL、サポートされている操作を記述します。

スキーマは自動生成されたドキュメントとして有用なツールになり得ます。また、APIとやり取りする動的なクライアントライブラリを使用する上でも役立つでしょう。

## Core API  
RESTフレームワークはスキーマのサポートにCore APIを使用しています。

Core APIはAPIを記述するための仕様であり、ドキュメントでもあります。利用可能なエンドポイントと、APIが公開する有効な内部表現形式を提供します。環境はサーバ、クライアントを問いません。

サーバサイドでCore APIを使用すると、広範囲のスキーマおよびハイパーメディア形式へのレンダリングをサポートすることができます。

クライアントサイドでCore APIを使用する場合は、サポートされているスキーマおよびハイパーメディア形式を公開するAPIと対話する、動的駆動クライアントライブラリとなります。

## スキーマを追加する  
RESTフレームワークは、明示的なスキーマビューの定義、スキーマの自動生成のどちらもサポートしています。今回はViewSetとRouterを使用しているので、スキーマを自動生成するのも簡単です。

APIスキーマを組み込むには、Pythonパッケージcoreapiをインストールする必要があります。

```
$ pip install coreapi
```

これでAPIにスキーマを組み込むことができるようになりました。URL設定に自動生成されたスキーマビューを組み込んでみましょう。

```
from rest_framework.schemas import get_schema_view

schema_view = get_schema_view(title='Pastebin API')

urlpatterns = [
    url(r'^schema/$', schema_view),
    ...
]
```

ブラウザでAPIルートの[エンドポイント](http://127.0.0.1:8000/schema/)にアクセスすると、オプションにcorejson形式が追加されたのがわかるはずです。

Acceptヘッダに目的のコンテントタイプを指定することで、コマンドラインからスキーマを要求することも可能です。

```
$ http http://127.0.0.1:8000/schema/ Accept:application/coreapi+json
HTTP/1.0 200 OK
Allow: GET, HEAD, OPTIONS
Content-Type: application/coreapi+json

{
    "_meta": {
        "title": "Pastebin API"
    },
    "_type": "document",
    ...
```

デフォルトの出力はCore JSONエンコードスタイルを使用するようになっています。

他のスキーマ形式としては、Open API（旧Swagger）などもサポートされています。

## コマンドラインクライアントを使う  
APIがスキーマのエンドポイントを公開するようになったので、動的クライアントライブラリを使ってAPIと対話することができます。実証してみましょう。Core APIコマンドラインクライアントを使います。

コマンドラインクライアントはcoreapi-cliパッケージに含まれています。

```
$ pip install coreapi-cli
```

さて、使えるようになったかコマンドラインから確認してみましょう。

```
$ coreapi
Usage: coreapi [OPTIONS] COMMAND [ARGS]...

  Command line client for interacting with CoreAPI services.

  Visit http://www.coreapi.org for more information.

Options:
  --version  Display the package version number.
  --help     Show this message and exit.

Commands:
...
```

まずはコマンドラインクライアントを使って、APIスキーマを読み込んでみます。

```
$ coreapi get http://127.0.0.1:8000/schema/
<Pastebin API "http://127.0.0.1:8000/schema/">
    snippets: {
        highlight(id)
        list()
        read(id)
    }
    users: {
        list()
        read(id)
    }
```

まだ認証処理を行っていないため、APIへのアクセス許可設定に基づき、読み取り専用のエンドポイントしか見ることができません。

コマンドラインクライアントを使って、既存のSnippetのリストを取得してみましょう。

```
$ coreapi action snippets list
[
    {
        "url": "http://127.0.0.1:8000/snippets/1/",
        "id": 1,
        "highlight": "http://127.0.0.1:8000/snippets/1/highlight/",
        "owner": "lucy",
        "title": "Example",
        "code": "print('hello, world!')",
        "linenos": true,
        "language": "python",
        "style": "friendly"
    },
    ...
```

APIエンドポイントには名前付きパラメーターが必要な場合があります。例えば特定のSnippetのハイライトHTMLを取得したい場合、IDを指定しなければなりません。

```
$ coreapi action snippets highlight --param id=1
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">

<html>
<head>
  <title>Example</title>
  ...
```

## クライアントから認証する  
Snippetを作成・編集・削除するためには、有効なユーザーとして認証しなくてはなりません。今回はBasic認証を使いましょう。

下記の<username>と<password>を実際のものに置き換えてください。

```
$ coreapi credentials add 127.0.0.1 <username>:<password> --auth basic
Added credentials
127.0.0.1 "Basic <...>"
```

ここでもう一度スキーマを取得してみましょう。利用可能なアクションのすべてが表示されるはずです。

```
$ coreapi reload
Pastebin API "http://127.0.0.1:8000/schema/">
    snippets: {
        create(code, [title], [linenos], [language], [style])
        delete(id)
        highlight(id)
        list()
        partial_update(id, [title], [code], [linenos], [language], [style])
        read(id)
        update(id, code, [title], [linenos], [language], [style])
    }
    users: {
        list()
        read(id)
    }
```

これでエンドポイントと対話できるようになりました。例として、新たにSnippetを作成してみます。

```
$ coreapi action snippets create --param title="Example" --param code="print('hello, world')"
{
    "url": "http://127.0.0.1:8000/snippets/7/",
    "id": 7,
    "highlight": "http://127.0.0.1:8000/snippets/7/highlight/",
    "owner": "lucy",
    "title": "Example",
    "code": "print('hello, world')",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

Snippetの削除も可能です。

```
$ coreapi action snippets delete --param id=7
```

コマンドラインクライアントに限らず、開発者であればクライアントライブラリを使用してAPIとやり取りすることもできます。Pythonのクライアントライブラリはこの先駆けであり、近い将来にはJavascriptのクライアントライブラリもリリースされる予定です。

スキーマ生成のカスタマイズや、クライアントライブラリであるCore APIの使用方法については、完全なドキュメントを参照してください。

## これまでを振り返って  
わずかなコードを書いただけですが、今や完全なpastebin Web APIを実装できました。すべてWebブラウザで閲覧可能で、スキーマ駆動クライアントライブラリを備え、認証処理や各オブジェクトの閲覧権限も完備し、複数のレンダリング形式をサポートしています。

これまでひとつひとつ処理を設計する中で、何かをカスタマイズする必要に迫られたとき、徐々に通常のDjangoビューらしくなるよう進化していく姿を見てきました。

GitHubにある[チュートリアルコード](https://github.com/encode/rest-framework-tutorial)を見直したり、ライブサンプルである[sandbox](http://restframework.herokuapp.com/)でいろいろ試すこともできます。

## 今回はここまで  
ここがチュートリアルの最終到達地点です。RESTフレームワークプロジェクトに貢献したい場合は、以下3つのスタート地点を紹介しておきます。

[GitHub](https://github.com/encode/django-rest-framework)でレビューしたり、Issueを投稿したり、Pull Requestを作成したりする
[REST framework discussion group](https://groups.google.com/forum/?fromgroups#!forum/django-rest-framework)に参加してコミュニティに貢献する
作者の[Twitter](https://twitter.com/_tomchristie)をフォローしてこんにちはする
それでは、良きRESTライフを。
