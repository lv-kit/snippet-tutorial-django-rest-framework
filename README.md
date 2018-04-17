# イントロダクション  
このチュートリアルでは、シンプルなpastebin形式のコードハイライトWebAPIを作っていきます。その過程でRESTフレームワークを形作る様々な要素を紹介しつつ、それぞれの機能がどう組み合わされているのか、包括的な理解をしてもらえればと思います。

チュートリアルといってもかなり長いので、読み始める前にクッキーとお好みのお酒などを用意したほうがいいかも知れません。概要とちゃちゃっと知りたいだけなら、[クイックスタート](http://www.django-rest-framework.org/tutorial/quickstart/)を読んだほうがいいです。

---

Note: このチュートリアルのコードは、GitHubの[tomchristie/rest-framework-tutorial](https://github.com/encode/rest-framework-tutorial)リポジトリにあります。完全な実装については、テスト用の[sandbox](http://restframework.herokuapp.com/)も兼ねてこちらを参照のこと。

---

## 新しい環境を作る  
何をするにもまずは virtualenv を使って新しい仮想環境を作りましょう。これでどんなパッケージをインストールしても、他のプロジェクトに影響することはなくなります。
```
virtualenv env
source env/bin/activate
```

仮想環境内に入ったので、パッケージを好きにインストールできます。

```
pip install django
pip install djangorestframework
pip install pygments  # コードハイライトに使用する
```

Note: virtualenv環境を終了するには deactivate と入力すればいつでも抜けることができます。詳細はvirtualenvのドキュメントを参照してください。

## ことはじめ  
さて、コーディングを始める準備ができました。 まずは作業する新しいプロジェクトを作成します。

```
cd ~
django-admin.py startproject tutorial
cd tutorial
```

さらに、シンプルなWeb APIの実装に必要なアプリケーションを作成します。

```
python manage.py startapp snippets .
```

新しいアプリケーション snippets と rest_framework を INSTALLED_APPS に追加する必要があります。 'tutorial/settings.py' ファイルを編集しましょう。

```
INSTALLED_APPS = (
    ...
    'rest_framework',
    'snippets.apps.SnippetsConfig',
    )
```

Django <1.9を使用している場合は、 snippets.apps.SnippetsConfig を snippets に置き換える必要があります。
よろしいですか？ いざ参りましょう。

## モデルを作成する  
このチュートリアルでは、コードスニペットの格納に使用するシンプルな Snippet モデルを作成するところから始めます。snippets/models.pyファイルを編集しましょう。（優れたプログラミングにはコメントが不可欠です。リポジトリにあるチュートリアルのコードにはコメントが含まれていますが、ここではコードそのものに焦点を当てるため省略しています）

```
from django.db import models
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles

LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted((item, item) for item in get_all_styles())


class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

    class Meta:
        ordering = ('created',)
```

また、snippetモデルのマイグレーションを作成し、データベースと同期させる必要があります。

```
python manage.py makemigrations snippets
python manage.py migrate
```

## Serializerクラスの作成  
Web APIを実装する第一歩として、snippetインスタンスを json などの形式にシリアライズしたり、デシリアライズする手段を提供する必要があります。これを行うには、Djangoのフォームによく似たシリアライザを定義します。snippetsディレクトリにserializers.pyという名前のファイルを作成し、以下のように記述しましょう。

```
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

    def create(self, validated_data):
        """
        Create and return a new `Snippet` instance, given the validated data.
        """
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        Update and return an existing `Snippet` instance, given the validated data.
        """
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance
```

シリアライザクラスの先頭では、シリアライズ／デシリアライズされるフィールドを定義しています。create()およびupdate()メソッドには、serializer.save()を呼び出すときにどれだけの情報を持ったインスタンスが作成、変更されるかを定義します。

シリアライザクラスはDjangoのFormクラスと非常によく似ており、required、max_length、defaultなど、様々なフィールドに対する検証フラグを設定することが出来ます。

フィールドのフラグは、HTML形式にレンダリングされたときなどの特定の状況で、シリアライザをどのように表示するかを設定することもできます。上記の{'base_template': 'textarea.html'}フラグは、DjangoのFormクラスでwidget=widgets.Textareaを使用するのと同じです。チュートリアルの後半で説明しますが、ブラウザでAPIをどのように表示するか設定したい場合、特に役立つでしょう。

実はModelSerializerクラスを使っても同様のことが可能です（後に説明します）が、今はシリアライザを明示的に定義しておきましょう。


## シリアライザを使ってみる  
先に進む前に、新たなSerializerクラスの使い方に慣れておきましょう。Djangoシェルに飛び込みます。

```
python manage.py shell
```

いくつかインポートが完了したら、コードスニペットを作ってみます。

```
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser

snippet = Snippet(code='foo = "bar"\n')
snippet.save()

snippet = Snippet(code='print "hello, world"\n')
snippet.save()
```

これでsnippetインスタンスで遊ぶことができます。これらのインスタンスのうち、ひとつをシリアライズする方法を見ていきましょう。

```
serializer = SnippetSerializer(snippet)
serializer.data
# {'id': 2, 'title': u'', 'code': u'print "hello, world"\n', 'linenos': False, 'language': u'python', 'style': u'friendly'}
```

この時点で、モデルインスタンスがPythonのネイティブデータ型に変換されました。シリアライズを完了するには、このデータをjson形式に落とし込みます。

```
content = JSONRenderer().render(serializer.data)
content
# '{"id": 2, "title": "", "code": "print \\"hello, world\\"\\n", "linenos": false, "language": "python", "style": "friendly"}'
```

デシリアライズも同様です。まずストリームをPythonのネイティブデータ型にパースして、

```
from django.utils.six import BytesIO

stream = BytesIO(content)
data = JSONParser().parse(stream)
```

このネイティブデータ型を、完全なオブジェクトインスタンスに復元します。

```
serializer = SnippetSerializer(data=data)
serializer.is_valid()
# True
serializer.validated_data
# OrderedDict([('title', ''), ('code', 'print "hello, world"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])
serializer.save()
# <Snippet: Snippet object>
```

フォームの作成とAPIの作成はよく似ていますよね。シリアライザを使用するビューを作成し始めると、もっと似ていることに気がつくはずです。

モデルインスタンスの代わりにクエリセットをシリアライズすることも出来ます。シリアライザの引数にmany=Trueフラグを追加するだけです。


```
serializer = SnippetSerializer(Snippet.objects.all(), many=True)
serializer.data
# [OrderedDict([('id', 1), ('title', u''), ('code', u'foo = "bar"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 2), ('title', u''), ('code', u'print "hello, world"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 3), ('title', u''), ('code', u'print "hello, world"'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])]
```

## ModelSerializersを使う  
これまでのSnippetSerializerクラスには、Snippetモデルに含まれている情報の多くが二重に記述されています。コードをもうちょっと簡潔に出来れば文句はないのですが。

DjangoがFormクラスとModelFormクラスを提供するのと同じように、RESTフレームワークはSerializerクラスとModelSerializerクラスを提供します。

ModelSerializerクラスを使ってシリアライザをリファクタリングしてみましょう。もう一度snippets/serializers.pyファイルを開き、SnippetSerializerクラスを以下のように書き換えます。

```
class SnippetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Snippet
        fields = ('id', 'title', 'code', 'linenos', 'language', 'style')
```

シリアライザが持つ優れた特徴のひとつは、シリアライザインスタンスの持つすべてのフィールドを調査し、形式を出力できることです。python manage.py shellでDjangoシェルを開き、試してみましょう。

```
from snippets.serializers import SnippetSerializer
serializer = SnippetSerializer()
print(repr(serializer))
# SnippetSerializer():
#    id = IntegerField(label='ID', read_only=True)
#    title = CharField(allow_blank=True, max_length=100, required=False)
#    code = CharField(style={'base_template': 'textarea.html'})
#    linenos = BooleanField(required=False)
#    language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
#    style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...
```

覚えておくべきは、ModelSerializerは魔法じみたことなど一切行わないということです。以下はシリアライザクラスを作成するためのショートカットに過ぎません。

* フィールドセットの自動判別  
* create()およびupdate()メソッドのシンプルなデフォルト実装  

## Serializerを使って通常のDjangoビューを書く  
新しいSerializerクラスを使ってAPIビューを作成する方法を見てみましょう。現時点ではRESTフレームワークの他の機能は使用しませんので、通常のDjangoビューとして作成します。

snippets/views.pyファイルを開き、以下を追加します。

```
from django.http import HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
```

APIのルートでは、既存のすべてのsnippetの一覧表示とsnippetの新規作成をサポートします。

```
@csrf_exempt
def snippet_list(request):
    """
    List all code snippets, or create a new snippet.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return JsonResponse(serializer.data, safe=False)
    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data, status=201)
        return JsonResponse(serializer.errors, status=400)
```

CSRFトークンを持たないクライアントからビューにPOSTできるようにするため、ビューにcsrf_exemptを印を付ける必要があります。普通はこんなことしたくないでしょうし、RESTフレームワークのビューではもっとスマートな動作が行われますが、とりあえず今はチュートリアルなので我慢してください。

個々のsnippetに対応するビューも必要です。snippetの取得、更新、削除に使われます。

```
@csrf_exempt
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return JsonResponse(serializer.data)

    elif request.method == 'PUT':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data)
        return JsonResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        snippet.delete()
        return HttpResponse(status=204)
```

最後にこれらのビューを関連付けましょう。snippets/urls.pyファイルを作成します。

```
from django.conf.urls import url
from snippets import views

urlpatterns = [
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
]
```

また、ルートのurlconfであるtutorial/urls.pyファイルと関連付け、snippetアプリケーションのURLを含むよう設定する必要があります。

```
from django.conf.urls import url, include

urlpatterns = [
    url(r'^', include('snippets.urls')),
]
```

現時点で、適切に処理していないギリギリアウトな実装が存在することは否定できません。仮に不正なjsonが送信されたり、ビューがサポートしていないメソッドでリクエストが飛んできた場合、500 "server error"レスポンスを返してしまいます。今はとりあえず動作すれば良しとしましょう。

## Web APIテスト: 最初の一歩  
これで、snippetサービスのサンプルサーバを立ち上げることができます。

シェルを終了して、

```
quit()
```

Djangoのdevelopmentサーバを起動します。
```
python manage.py runserver

Validating models...

0 errors found
Django version 1.11, using settings 'tutorial.settings'
Development server is running at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

別の端末ウィンドウでサーバをテストできます。

curlやhttpieを用いてAPIをテストしてみます。HttpieはPythonで書かれたユーザフレンドリなhttpクライアントです。インストールしましょう。

pip経由でインストールできます:

```
pip install httpie
```

これでやっと、すべてのsnippetのリストを取得することができます。

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

またはIDを指定して特定のsnippetを取得することもできます。

```
http http://127.0.0.1:8000/snippets/2/

HTTP/1.1 200 OK
...
{
  "id": 2,
  "title": "",
  "code": "print \"hello, world\"\n",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}
```

同様に、WebブラウザでこれらのURLにアクセスし、同じjsonを表示することもできます。

## まとめ  
何とかここまでやってきました。DjangoのフォームAPIによく似たシリアライズAPIが使えるようになり、通常のDjangoビューも使えます。

現在のAPIビューでは、jsonレスポンスを提供する以外に特別なことは何もしていません。さらにクリーンアップしてエラーハンドリングしたいギリギリの実装もありますが、少なくともWeb APIとして機能することはしています。
