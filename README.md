# 第06回: ViewSetとRouter  
RESTフレームワークにはViewSetと呼ばれる抽象構造が含まれており、開発者がAPIの状態や操作のモデリングに集中できるよう、URLを一般的な規約に基いたものに自動構築させることができます。

ViewSetクラスはViewクラスとほとんど同じですが、getやputといったメソッドハンドラではなく、readやupdateなどのオペレーションを提供します。

ViewSetクラスが一連のメソッドハンドラに割り当てられるのはビューにインスタンス化される瞬間のみで、通常これは複雑なURLconfを定義するRouterクラスを介して行われます。

## ViewSetを使うようリファクタリングする  
現在のビューをViewSetに落とし込んでみましょう。

まずはUserListとUserDetailを単一のUserViewSetにリファクタリングします。2つのビューを削除し、1つのクラスに置き換えましょう。

```
from rest_framework import viewsets

class UserViewSet(viewsets.ReadOnlyModelViewSet):
    """
    This viewset automatically provides `list` and `detail` actions.
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

ここではReadOnlyModelViewSetクラスを使って、デフォルトの「読み取り専用」オペレーションを自動的に定義しています。querysetとserializer_class属性は通常のViewを使っていたときと同じく設定していますが、同じ情報を2つのクラスそれぞれに提供する必要はなくなりました。

次はSnippetList, SnippetDetail, SnippetHighlightクラスを置き換えます。3つのビューを削除して、やはり1つのクラスに書き直しましょう。

```
from rest_framework.decorators import detail_route
from rest_framework.response import Response

class SnippetViewSet(viewsets.ModelViewSet):
    """
    This viewset automatically provides `list`, `create`, `retrieve`,
    `update` and `destroy` actions.

    Additionally we also provide an extra `highlight` action.
    """
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                          IsOwnerOrReadOnly,)

    @detail_route(renderer_classes=[renderers.StaticHTMLRenderer])
    def highlight(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

こちらではModelViewSetクラスを使うことで、デフォルトの「読み書き可能」オペレーションを網羅しています。

@detail_routeデコレータを併用してhighlightという名前のカスタムアクションを作成しました。このデコレータを使うことで、標準のcreate, update, deleteスタイルに合致しないカスタムエンドポイントを定義することができます。

@detail_routeデコレータを使用するカスタムアクションは、デフォルトでGETリクエストに対応します。POSTリクエストに対応させたい場合はmethods引数を指定してください。

カスタムアクションのURLには、デフォルトではメソッド名そのものが割り当てられます。URLパターンの構築方法を変更する場合は、デコレータのキーワード引数にurl_pathを指定してください。

## ViewSet <-> URLの明示的な割り当て  

URLconfを定義すると、ハンドラメソッドはアクションにのみ割り当てられます。 内部的には何が行われているのでしょうか。とりあえず、ViewSetを明示的にビューに変換してみましょう。

urls.pyファイルで、ViewSetクラスを実際のビューに割り当てます。

```
from snippets.views import SnippetViewSet, UserViewSet, api_root
from rest_framework import renderers

snippet_list = SnippetViewSet.as_view({
    'get': 'list',
    'post': 'create'
})
snippet_detail = SnippetViewSet.as_view({
    'get': 'retrieve',
    'put': 'update',
    'patch': 'partial_update',
    'delete': 'destroy'
})
snippet_highlight = SnippetViewSet.as_view({
    'get': 'highlight'
}, renderer_classes=[renderers.StaticHTMLRenderer])
user_list = UserViewSet.as_view({
    'get': 'list'
})
user_detail = UserViewSet.as_view({
    'get': 'retrieve'
})
```

各ViewSetクラスからどのように複数のビューを切り出しているのか見てみると、httpメソッドを各ビューの対応するアクションに割り当てていることがわかります。

これでリソースが実際のビューに割り当てられました。いつも通りビューをURLconfに登録しましょう。

```
urlpatterns = format_suffix_patterns([
    url(r'^$', api_root),
    url(r'^snippets/$', snippet_list, name='snippet-list'),
    url(r'^snippets/(?P<pk>[0-9]+)/$', snippet_detail, name='snippet-detail'),
    url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', snippet_highlight, name='snippet-highlight'),
    url(r'^users/$', user_list, name='user-list'),
    url(r'^users/(?P<pk>[0-9]+)/$', user_detail, name='user-detail')
])
```

## Routerを使う  
ViewクラスではなくViewSetクラスを使用しているため、実は手動でURLを設計する必要はありません。リソースをビュー・URLに関連付けるための規約はRouterクラスを使って自動的に処理させることができます。適切なViewSetをRouterに登録してしまえば、あとはお任せです。

こちらがスリムになったurls.pyファイルです。

```
from django.conf.urls import url, include
from snippets import views
from rest_framework.routers import DefaultRouter

# Create a router and register our viewsets with it.
router = DefaultRouter()
router.register(r'snippets', views.SnippetViewSet)
router.register(r'users', views.UserViewSet)

# The API URLs are now determined automatically by the router.
# Additionally, we include the login URLs for the browsable API.
urlpatterns = [
    url(r'^', include(router.urls)),
    url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```

ViewSetをRouterに登録するのはurlpatternを定義するのに似ており、ビューのURLプレフィックスとViewSetそのものの2つの引数が含まれています。

DefaultRouterクラスは自動的にAPIルートビューを作成するので、viewsモジュールからapi_rootメソッドを削除することができます。

## View 対 ViewSet - どっちがいいの？  
ViewSetを使えば非常に便利な抽象構造を手に入れられます。URLの規約がAPI全体を通して一貫していることが保証され、書くコードの量は最小限になり、URLconfの仕様とにらめっこするのではなく、APIが提供する処理や形式だけに集中することができます。

しかし、それが常に正しいアプローチであるとは限りません。関数ベースのビューとクラスベースのビュー、どちらを使うべきかという問題にも似ていて、いずれもトレードオフを考慮する必要があります。ViewSetを使う場合、ビューを個別に作成するよりも明示性で劣るとも言えます。

[チュートリアル 第07回](http://sandmark.hateblo.jp/entry/2017/10/06/190611)ではAPIスキーマを追加する方法や、構築したAPIに対するクライアントライブラリ・コマンドラインツールからのアクセス方法を見ていきます。

