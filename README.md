# 第02回 「リクエストとレスポンス」  
今回からRESTフレームワークの中核部分に触れていきます。まずは重要な構成要素を紹介しましょう。  

## Requestオブジェクト  
RESTフレームワークは通常のHttpRequestを拡張し、より柔軟なリクエストパースをサポートするRequestオブジェクトを提供しています。Requestオブジェクトのコア機能はrequest.data属性で、request.POSTに似ていますが、Web APIを扱う上で使いやすくなるようチューニングされています。  

```
request.POST  # フォームデータのみ処理する。'POST'メソッドでしか動作しない。
request.data  # 任意のデータを処理する。'POST', 'PUT', 'PATCH'メソッドで動作する。
```

## Responseオブジェクト  
同様に、RESTフレームワークはResponseオブジェクトも提供します。これはTemplateResponseオブジェクトの一種であり、レンダリングされていないコンテントを受け取り、コンテントネゴシエーションを使用して、クライアントに正しいコンテントタイプを判定します。  

```
return Response(data)  # クライアントにリクエストされたコンテントタイプでレンダリングする
```

## ステータスコード  
ビューの中で数値のHTTPステータスコードを使っても、常にわかりやすいコードになるわけではなく、エラーコードが間違っていることに気が付かないのはザラにあります。RESTフレームワークはstatusモジュールのHTTO_400_BAD_REQUESTのような、各ステータスコードに対して明白な識別子を提供します。数値の識別子を使うのではなく、こちらを使用するのをお勧めします。

## APIビューをラップする  
RESTフレームワークでは、APIビューの作成に便利なラッパーを2つ提供されています。

1. 関数ベースのビューを扱うための@api_viewデコレータ。
2. クラスベースのビューを扱うためのAPIViewクラス。
これらのラッパーには、ビューにRequestインスタンスを確実に受け取り、コンテンツネゴシエーションを実行できるようにResponseオブジェクトにコンテキストを追加するなどの機能が提供されています。

また、然るべき場合に405 Method Not Allowedレスポンスを返すような動作を提供し、誤った入力を伴うrequest.dataにアクセスした際にParseError例外を扱います。

## 全部ひっくるめて  
では、これらの新しいコンポーネントを使ってビューを書いてみましょう。

もはやviews.pyにはJSONResponseクラスは必要ないので消してしまいます。これによりビューをちょっとだけリファクタリングすることができます。

```
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer


@api_view(['GET', 'POST'])
def snippet_list(request):
    """
    List all code snippets, or create a new snippet.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

前回よりも洗練されたインスタンスビューになりました。やや簡潔になり、Form APIを使っている場合と非常によく似ています。名前付きステータスコードにより、レスポンスの意味が明確になっています。

こちらはviews.pyモジュールの単体snippetビューの定義です。

```
@api_view(['GET', 'PUT', 'DELETE'])
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

身近に感じられることでしょう。これは通常のDjangoビューと比較しても違いはそれほどありません。

特定のコンテントタイプへのリクエストやレスポンスを明示的に関連付ける必要はもうありません。request.dataはjsonリクエストを扱うことも出来ますし、他のフォーマットも扱うことも出来ます。同様にレスポンスオブジェクトをデータと一緒に返していますが、これもRESTフレームワークが正しいコンテントタイプでレンダリングしてくれます。

## URLにオプションとしてフォーマット接尾子を追加する  
レスポンスがひとつのコンテントタイプに縛られなくなったということで、APIのエンドポイントにフォーマット接尾子を追加しましょう。フォーマット接尾子を使用すると、指定されたフォーマットを明示的に参照するURLが提供され、APIがhttp://example.com/api/items/4.jsonなどのURLを処理できるようになります。

まずは両方のビューにキーワード引数formatを追加します。


```
def snippet_list(request, format=None):
```

```
def snippet_detail(request, pk, format=None):
```

少しだけurls.pyファイルをいじって、既存のURLに加えてformat_suffix_patternsを追加してください。

```
from django.conf.urls import url
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)$', views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

拡張URLパターンは必ずしも追加する必要はありませんが、特定フォーマットを参照する方法として、シンプルかつクリーンな手段を提供します。

## 見た目はどんな感じ？  
[チュートリアル 第01回](http://sandmark.hateblo.jp/entry/2017/09/30/160945)で行ったように、コマンドラインからAPIをテストしてみましょう。いずれも同じように動作していますが、無効なリクエストを行った場合、エラー処理がちゃんと動作しています。

前回同様、すべてのsnippetのリストを取得するには以下のようにします。


```
http http://127.0.0.1:8000/snippets/

HTTP/1.1 200 OK
...
[
  {
    "id": 1,
    "title": "",
    "code": "foo = \"bar\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  },
  {
    "id": 2,
    "title": "",
    "code": "print \"hello, world\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  }
]
```

Acceptヘッダを使って、返されるレスポンスのフォーマットを指定することができます。

```
http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON
http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML
```

またはフォーマット接尾子を追加することもできます。

```
http http://127.0.0.1:8000/snippets.json  # JSON suffix
http http://127.0.0.1:8000/snippets.api   # Browsable API suffix
```

同様にContent-Typeヘッダを使うことで、送信するリクエストのフォーマットを制御できます。

```
# フォームデータを使用したPOST
http --form POST http://127.0.0.1:8000/snippets/ code="print 123"

{
  "id": 3,
  "title": "",
  "code": "print 123",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}

# JSONを使用したPOST
http --json POST http://127.0.0.1:8000/snippets/ code="print 456"

{
    "id": 4,
    "title": "",
    "code": "print 456",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

上記httpリクエストに--debug引数を追加すると、リクエストヘッダにあるリクエストタイプを見ることができます。

ではhttp://127.0.0.1:8000/snippets/にアクセスして、WebブラウザでAPIを開いてみましょう。

## ブラウザビリティ  
APIはクライアントのリクエストに基いてレスポンスのコンテントタイプを選択するため、リソースがWebブラウザから要求された場合、デフォルトではHTML形式のリソースを返します。これにより、APIは完全にWebブラウズ可能なHTMLフォーマットを返すことができます。

Webブラウズ可能なAPIを使用するメリットとして、開発速度と使い勝手が大幅に向上します。また開発中のAPIを参照・使用したいと考える他の開発者にとっても、参入障壁が劇的に低くなります。

Webブラウズ可能なAPIの詳細とカスタマイズ方法についてはbrowsable apiトピックを参照してください。

## 次の目標  
チュートリアル 第03回からはクラスベースのビューを使い、ジェネリックビューを使うことで記述コードの量を減らす方法を見ていきます。
