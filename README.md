# 第03回: クラスベースのビュー  
関数ベースのビューではなく、クラスベースのビューを使ってAPIビューを記述することもできます。今回説明するように、この手法は共通の機能を再利用できる強力なデザインパターンであり、コードをDRY（日本語記事）に保つことができます。

## クラスベースのビューを使ってAPIを書き直す  
まずはルートのビューをクラスベースのビューとして書き直します。そのためにviews.pyを少しリファクタリングします。

```
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from django.http import Http404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status


class SnippetList(APIView):
    """
    List all snippets, or create a new snippet.
    """
    def get(self, request, format=None):
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

順調ですね。前回とよく似ていますが、各種HTTPメソッドをうまく分離しています。views.pyで、同じくインスタンスビューも書き換える必要があります。

```
class SnippetDetail(APIView):
    """
    Retrieve, update or delete a snippet instance.
    """
    def get_object(self, pk):
        try:
            return Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            raise Http404

    def get(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    def put(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        snippet = self.get_object(pk)
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

いい感じになりました。とはいえ、まだ関数ベースのビューに近しいものがあります。

クラスベースのビューを使用するようになったため、今度はurls.pyも少しリファクタリングしなければなりません。

```
from django.conf.urls import url
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    url(r'^snippets/$', views.SnippetList.as_view()),
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.SnippetDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

これで完了です。開発サーバを立ち上げても、すべて以前と同じく動作するはずです。

## Mixinを使う  
クラスベースのビューを使用する大きなメリットのひとつに、再利用可能な振る舞いを簡単に作成できるという点があります。

これまでに使ってきた作成／取得／更新／削除の操作は、モデルから提供されるAPIをほぼそのまま呼び出していました。こうした共通の動作はすでに、RESTフレームワークのmixinクラスで実装されています。

mixinクラスを使ってビューを作成する方法を見てみましょう。views.pyモジュールを再掲します。

```
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import mixins
from rest_framework import generics

class SnippetList(mixins.ListModelMixin,
                  mixins.CreateModelMixin,
                  generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

何が何だかわかりませんね。踏み込んで調べてみましょう。上記ではGenericAPIViewを使用し、ListModelMixinとCreateModelMixinを追加しています。

ベースクラスはコア機能を提供するもので、mixinクラスからは.list()と.create()アクションを提供されます。getメソッドとpostメソッドを明示的に適切なアクションへ割り当てているわけです。紐解いてみると単純でしたね。

```
class SnippetDetail(mixins.RetrieveModelMixin,
                    mixins.UpdateModelMixin,
                    mixins.DestroyModelMixin,
                    generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```

こちらも似ています。ここでもGenericAPIViewクラスを使用してコア機能を利用つつ、mixinを追加して.retrieve(), .update(), .destroy()アクションを提供しています。

## ジェネリッククラスベースビュー  
mixinクラスを使うことで、以前よりコードの量が削減されたビューを書くことに成功しました。しかしさらなる進化の可能性があるのです。RESTフレームワークは、すでにmixinされたジェネリックビューのセットを提供しています。これを使ってviews.pyモジュールをもっとスリムにすることができます。

```
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import generics


class SnippetList(generics.ListCreateAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer


class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
```

衝撃的なほど簡潔になりました。Djangoらしいだけでなく、コードの見た目もバッチリかつクリーン。しかもこれが無料で手に入る時代なのです。

ではチュートリアル 第04回に進みましょう。次回はAPIの認証とアクセス制御をどのように処理していくのかを見ていきます。
