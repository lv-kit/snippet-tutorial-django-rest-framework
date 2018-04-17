# 第05回: リレーションシップとハイパーリンクAPI  
現時点のAPIでは、リレーションシップがプライマリキーで表現されています。今回のチュートリアルでは、キーの代わりにハイパーリンクを使用することで、APIの結合性と見やすさを改善していきます。

## APIルート用にエンドポイントを作成する  
今のところ 'snippets' と 'users' のエンドポイントはありますが、APIへのエントリポイントはひとつもありません。実装してみましょう。これまでに見てきた通常の関数ベースのビューと@api_viewデコレータを使用します。snippets/views.pyに以下を追加してください。

```
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.reverse import reverse


@api_view(['GET'])
def api_root(request, format=None):
    return Response({
        'users': reverse('user-list', request=request, format=format),
        'snippets': reverse('snippet-list', request=request, format=format)
    })
```

ここで注目すべき点が2つあります。ひとつは完全修飾URLを返すためにRESTフレームワークのreverse関数を使用していること。ふたつめは、URLパターンがわかりやすい名前で定義できるという点です（後のsnippets/urls.pyを参照）。

## ハイライトスニペット用にエンドポイントを作成する  
pastebinのAPIに足りないものは何かあるでしょうか。まだありますね。エンドポイントをハイライトするコードです。

他のAPIエンドポイントのようにJSONを使用するのはらしくありません。JSONではなくHTML表現で提供したいところです。RESTフレームワークが提供するHTML Rendererには、テンプレート経由でレンダリングされたHTMLを扱うものと、あらかじめレンダリングされたHTMLを扱うものの2種類があります。今回のエンドポイントに使いたいのは後者ですね。

コードハイライトビューを作る上で考慮しなければならないのは、既存のジェネリックビューをベースに書くことができないという点です。オブジェクトインスタンスを返すのではなく、オブジェクトインスタンスのプロパティを返す必要があるからです。

ジェネリックビューを使う代わりにベースクラスを使ってインスタンスを表現し、独自の.get()メソッドを定義します。snippets/views.pyに以下を追加してください。

```
from rest_framework import renderers
from rest_framework.response import Response

class SnippetHighlight(generics.GenericAPIView):
    queryset = Snippet.objects.all()
    renderer_classes = (renderers.StaticHTMLRenderer,)

    def get(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)
```

いつも通り、作成した新しいビューをURLconfに追加する必要があります。 新たなAPIルートのURLパターンをsnippets/urls.pyに追加しましょう。

```
url(r'^$', views.api_root),
```

それからSnippetHiglightのURLパターンも追加します。

```
url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', views.SnippetHighlight.as_view()),
```

## APIのハイパーリンク  
エンティティ間のリレーションシップの扱いは、Web APIを設計する中でも頭を悩まされることが多い事柄です。リレーションシップの表現手法には、それぞれまったく異なるアプローチがあります。

- プライマリキーを使う
- エンティティ間をハイパーリンクさせる
- 関連エンティティ上に、識別用のユニークなslugフィールドを用意する
- 関連エンティティのデフォルト文字列表現を使う
- 親の表現の中に関連エンティティを入れ子にする
- その他
RESTフレームワークはいずれのスタイルもサポートしており、フォワード・リバースリレーションシップに適用させることも、汎用外部キーなどのカスタム管理に適用させることも可能です。

今回はエンティティ間でハイパーリンクさせるスタイルを採用してみましょう。下準備として、既存のModelSerializerではなくHyperlinkedModelSerializerを使ってシリアライザを拡張します。

HyperlinkedModelSerializerは、以下の点でModelSerializerとは異なります。

- デフォルトではidフィールドが含まれない
- HyperlinkedIdentityFieldを用いたurlフィールドが含まれる
- リレーションシップにはHyperlinkedRelatedFieldを用い、PrimaryKeyRelatedFieldは使わない
既存のシリアライザを書き直してハイパーリンクに対応させるのは簡単です。snippets/serializers.pyに以下を追加してください。

```
class SnippetSerializer(serializers.HyperlinkedModelSerializer):
    owner = serializers.ReadOnlyField(source='owner.username')
    highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

    class Meta:
        model = Snippet
        fields = ('url', 'id', 'highlight', 'owner',
                  'title', 'code', 'linenos', 'language', 'style')


class UserSerializer(serializers.HyperlinkedModelSerializer):
    snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail', read_only=True)

    class Meta:
        model = User
        fields = ('url', 'id', 'username', 'snippets')
```

ここで新たに'highlight'フィールドを追加しました。このフィールドはurlフィールドと同じ型ですが、URLパターンに'snippet-detail'ではなく'snippet-highlight'を指定しています。

これまでに接尾子付きのURL（'.json'など）をサポートしてきましたが、highlightフィールドはどんなフォーマットを指定されても'.html'接尾子が指定されたものとして返さなければならない点に注意してください。

## URLパターンの名前を確認する  
ハイパーリンクAPIを使用する場合、URLパターンの名前をしっかり把握しておく必要があります。どのURLパターンに名前を付けるべきなのか考えてみましょう。

- APIルートは'user-list'と'snippet-list'を参照している
- SnippetSerializerには'snippet-highlight'を参照するフィールドがある
- UserSerializerには'snippet-detail'を参照するフィールドがある
- SnippetSerializer, UserSerializerには'url'フィールドがあり、デフォルトで'{model_name}-detail'を参照する。この場合は'snippet-detail'と'user-detail'になっている
これらの名前をすべてURLconfに追加すると、最終的なsnippets/urls.pyファイルの中身はこんな感じになります。


```
from django.conf.urls import url, include
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

# API endpoints
urlpatterns = format_suffix_patterns([
    url(r'^$', views.api_root),
    url(r'^snippets/$',
        views.SnippetList.as_view(),
        name='snippet-list'),
    url(r'^snippets/(?P<pk>[0-9]+)/$',
        views.SnippetDetail.as_view(),
        name='snippet-detail'),
    url(r'^snippets/(?P<pk>[0-9]+)/highlight/$',
        views.SnippetHighlight.as_view(),
        name='snippet-highlight'),
    url(r'^users/$',
        views.UserList.as_view(),
        name='user-list'),
    url(r'^users/(?P<pk>[0-9]+)/$',
        views.UserDetail.as_view(),
        name='user-detail')
])

# Login and logout views for the browsable API
urlpatterns += [
    url(r'^api-auth/', include('rest_framework.urls',
                               namespace='rest_framework')),
]
```

## ページネーションを追加する  
UserとSnippetのリストビューは大量のインスタンスを返す可能性があります。実務では結果をページネートして、APIクライアントが個々のページにアクセスできるようにしておくべきでしょう。

tutorial/settings.pyファイルにわずかな変更を加えるだけで、デフォルトのリストにページネーションを適用することができます。以下の設定を追加してください。

```
REST_FRAMEWORK = {
    'PAGE_SIZE': 10
}
```

RESTフレームワークの設定はすべてREST_FRAMEWORKという名前の辞書に指定するため、他のプロジェクト設定と競合したり、見づらくなったりすることはありません。

必要に応じてページネーションのスタイルをカスタマイズすることもできますが、今回はデフォルト設定を使用します。

## ブラウザで確かめる  
ブラウザでAPIのURLを開いてみると、リンクを辿るだけでAPIを操作したり、移動したりすることができるようになったのがわかるはずです。

またsnippetインスタンスからhighlightフィールドがリンクされており、クリックするとHTML形式でハイライトされたコードが返ってきます。

チュートリアル 第06回ではViewSetsとRoutersを見ていきます。API設計に必要なコードの量を、これらを使って削減してみましょう。
