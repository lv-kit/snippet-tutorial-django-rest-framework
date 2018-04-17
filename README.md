# チュートリアル 第04回: 認証と許可  
これまで開発してきたAPIには、誰がコードスニペットを編集・削除できるのかという制限がありません。以下の点をはっきりさせる上で、もう少し高度な振る舞いをさせたいところです。

- コードスニペットは必ず製作者に関連付けられている。
- 認証されたユーザのみスニペットを作成できる。
- スニペットの製作者のみ更新・削除できる。
- 認証されていないリクエストにはすべて読み取り専用アクセスを提供する。

## モデルに情報を追加する  
Snippetモデルクラスにちょっと変更を加えてみます。 まずはフィールドを追加しましょう。ひとつはコードスニペットを作成したユーザを表すために使用されるもの、もうひとつはコードをハイライトしたときのHTML表現を格納するためのものです。

models.pyのSnippetモデルに以下の2つのフィールドを追加してください。

```
owner = models.ForeignKey('auth.User', related_name='snippets', on_delete=models.CASCADE)
highlighted = models.TextField()
```

またモデルが保存されるときに、コードハイライトライブラリであるpygmentsを使って、highlightedフィールドに値をセットする必要があります。

まずはimportしましょう。

```
from pygments.lexers import get_lexer_by_name
from pygments.formatters.html import HtmlFormatter
from pygments import highlight
```

そしてモデルクラスに.save()メソッドを追加します。

```
def save(self, *args, **kwargs):
    """
    Use the `pygments` library to create a highlighted HTML
    representation of the code snippet.
    """
    lexer = get_lexer_by_name(self.language)
    linenos = self.linenos and 'table' or False
    options = self.title and {'title': self.title} or {}
    formatter = HtmlFormatter(style=self.style, linenos=linenos,
                              full=True, **options)
    self.highlighted = highlight(self.code, lexer, formatter)
    super(Snippet, self).save(*args, **kwargs)
```

作業が完了したら、データベーステーブルを更新しなければなりません。 普通はマイグレーションを作成して行いますが、このチュートリアルではデータベースを削除して続行します。

```
rm -f tmp.db db.sqlite3
rm -r snippets/migrations
python manage.py makemigrations snippets
python manage.py migrate
```

異なるユーザを複数作成してAPIをテストすることもできます。一番簡単な方法はcreatesuperuserコマンドを使うことです。

```
python manage.py createsuperuser
```

## Userモデルにエンドポイントを追加する  
さて、今や複数のユーザが作業に参加してきましたので、UserをAPIで表現できるようにしたほうがいいでしょう。新しいシリアライザの作成は簡単です。serializers.pyに以下を追加してください。

```
from django.contrib.auth.models import User

class UserSerializer(serializers.ModelSerializer):
    snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

    class Meta:
        model = User
        fields = ('id', 'username', 'snippets')
```

snippetsはリバースリレーションシップであるため、ModelSerializerクラスを使うだけではデフォルトでは含まれません。そこで明示的にフィールドを追加する必要があります。

続いてviews.pyにビューを追加しましょう。Userの表現には読み取り専用のビューのみ提供したいので、ジェネリッククラスビューであるListAPIViewとRetrieveAPIViewを使います。

```
from django.contrib.auth.models import User


class UserList(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer


class UserDetail(generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

UserSerializerクラスのimportも忘れないでください。

```
from snippets.serializers import UserSerializer
```

最後に、これらのビューをURL confに設定してAPIに追加しなければなりません。urls.pyに以下のパターンを追加してください。

```
url(r'^users/$', views.UserList.as_view()),
url(r'^users/(?P<pk>[0-9]+)/$', views.UserDetail.as_view()),
```

## SnippetとUserの関連付け  
今のところコードスニペットを作成しても、スニペットを作成したユーザをスニペットインスタンスに関連付ける方法がありません。Userは受信するリクエストのプロパティに過ぎず、シリアライズされた形式で送信されるものではないのです。

これはsnippetビューの.perform_create()メソッドをオーバーライドすれば解決します。これにより、インスタンスのsave方法を管理し、受け取ったリクエスト、またはリクエストされたURLの暗黙的な情報を処理することができます。

SnippetListビュークラスに以下のメソッドを追加しましょう。

```
def perform_create(self, serializer):
    serializer.save(owner=self.request.user)
```

これでシリアライザのcreate()メソッドには、リクエストの検証済みデータとともに、追加されたownerフィールドが渡されるようになります。

## シリアライザを書き直す  
snippetは作成したユーザに関連付けられるようになったので、反映させるようにSnippetSerializerを書き直しましょう。serializers.pyのシリアライザ定義に以下のフィールドを追加します。

```
owner = serializers.ReadOnlyField(source='owner.username')
```

注意: インナークラスであるMetaにもownerフィールドを追加するのを忘れないでください。

このフィールドはとても興味深い動作をします。source引数はフィールドにセットされる値を制御するもので、シリアライズされたインスタンスの任意の属性を指定することができます。上記のようにドット記法で指定することもでき、この場合はDjangoのテンプレート言語で使用されているのと同じ方法で属性をトラバースします。

いま追加したのはCharFieldやBooleanFieldなどの型付きフィールドとはまったく違う、ReadOnlyFieldという型指定のないクラスです。ReadOnlyFieldは常に読み取り専用となり、シリアライズされた形式に使用されますが、デシリアライズされたときにモデルインスタンスを更新することはありません。今回のケースであればCharField(read_only=True)を使用することもできます。

## ビューに必要なパーミッションを追加する  
さて、コードスニペットはユーザを関連付けられました。そこで今度は認証されたユーザにのみ、コードスニペットの作成／更新／削除を許可したいところです。

RESTフレームワークには、特定のビューにアクセスできるユーザを制限するのに便利なパーミッションクラスが多く含まれています。今回求められているのはIsAuthenticatedOrReadOnlyです。認証されたリクエストには読み書きアクセス権を提供し、認証されていない場合は読み取り専用アクセス権を提供します。

まずはviewsモジュールに以下のimportを追加します。

```
from rest_framework import permissions
```

次に、SnippetListとSnippetDetailクラスの両方に以下のプロパティを追加します。

```
permission_classes = (permissions.IsAuthenticatedOrReadOnly,)
```

## ブラウザから見るAPIにログイン処理を追加する  
ここでブラウザを開いてAPIにアクセスすると、新しいコードスニペットが作成できなくなっていることがわかります。これを行うには、ユーザとしてログインできる環境を作らなければなりません。

プロジェクトトップにあるurls.pyファイルの URLconf を編集することで、ブラウザからAPI動作を確認するためのログインビューを追加することができます。

ファイルの先頭に以下のimportを追加します。

```
from django.conf.urls import include
```

そしてファイルの末尾に、ブラウザ用のログインビューとログアウトビューを定義するパターンを追加します。

```
urlpatterns += [
    url(r'^api-auth/', include('rest_framework.urls',
                               namespace='rest_framework')),
]
```

パターンのr'^api-auth/'部分は、実際に使用したいどんなURLでも構いません。唯一の制限は、includeされたURLがrest_frameworkというnamespaceを使わなければならないという点です。Django 1.9以降ではRESTフレームワークがnamespaceを設定するので、そのままでも大丈夫です。

ここでブラウザを開いてページを更新すると、ページの右上に「ログイン」というリンクが表示されるはずです。先ほど作成したユーザでログインすると、またコードスニペットを作成できるようになります。

コードスニペットを作成したら '/users/' エンドポイントに移動し、各ユーザの 'snippet' フィールドに、関連付けられたsnippetのIDのリストが表示されていることを確認してみてください。

## オブジェクトレベルパーミッション  
作成されたコードスニペットは誰にでも公開しておきたいところですが、更新／削除できるのは作成したユーザだけであるという点もまた忘れてはなりません。

これにはカスタムパーミッションを作成する必要があります。

snippetsアプリケーションに新しいファイルを追加しましょう。permissions.pyです。

```
from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Custom permission to only allow owners of an object to edit it.
    """

    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed to any request,
        # so we'll always allow GET, HEAD or OPTIONS requests.
        if request.method in permissions.SAFE_METHODS:
            return True

        # Write permissions are only allowed to the owner of the snippet.
        return obj.owner == request.user
```

これでsnippetインスタンスエンドポイントにカスタムパーミッションを追加できるようになりました。SnippetDetailビュークラスのpermission_classesプロパティを編集します。

```
permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                      IsOwnerOrReadOnly,)
```

IsOwnerOrReadOnlyクラスをインポートするのも忘れてはいけません。

```
from snippets.permissions import IsOwnerOrReadOnly
```

もう一度ブラウザを開きましょう。コードスニペットを作成したユーザと同じユーザでログインしていれば、該当snippetインスタンスのエンドポイントにのみ'DELETE'と'PUT'アクションが表示されるはずです。


## API経由の認証  
APIに対して一連のパーミッションが設定されているため、snippetを編集する場合はAPIリクエストの認証処理を行う必要があります。authentication classesは何一つ設定していないため、現在はデフォルトの設定であるSessionAuthenticationとBasicAuthenticationが適用されています。

Webブラウザを介してAPIとやり取りするなら、ログインもできますし、ブラウザのセッションがリクエストに必要な認証処理を行ってくれます。

ではプログラマティックにAPIとやり取りするにはどうすればいいのかというと、リクエストごとの認証情報を明示的に指定しなければなりません。

認証せずにsnippetを作成しようとすると、このようなエラーが返ってきます。

```
http POST http://127.0.0.1:8000/snippets/ code="print 123"

{
    "detail": "Authentication credentials were not provided."
}
```

先ほど作成したユーザの名前とパスワードを指定することで、リクエストを正常に完了することができます。

```
http -a tom:password123 POST http://127.0.0.1:8000/snippets/ code="print 789"

{
    "id": 1,
    "owner": "tom",
    "title": "foo",
    "code": "print 789",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

## まとめ  
今やこのWeb APIには、きわめて詳細に設定されたパーミッションが付与され、そのシステム上のコードスニペットや、その作成者たるユーザのエンドポイントとしての柔軟さも備えられています。

チュートリアル 第05回では、ハイライトされたsnippetのHTMLエンドポイントを作成することで全てを結びつけ、システム内のリレーションシップにハイパーリンクを用いることで、APIの結合性を高めていきます。
