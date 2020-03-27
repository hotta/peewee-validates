基本的なバリデーション
=======================

最も基本的な使い方は、以下のように validator クラスを使う方法です:

.. code:: python

    from peewee_validates import Validator, StringField, validate_not_empty

    class SimpleValidator(Validator):
        first_name = StringField(validators=[validate_not_empty()])

    validator = SimpleValidator()

これはある first_name というフィールドのデータをバリデート（正当であることを確認）したいことを表しています。

それぞれのフィールドは関連するデータ型を持っています。
このケースでは、StringField を使って入力データを強制的に ``str`` として扱います。

バリデータインスタンスの作成後、 ``validate()`` メソッドを呼んでバリデートしたいデータを渡します。
その戻り値は、すべてのバリデーションが成功したかどうかを表すブール値です。

バリデータはこの後 ``data`` と ``errors`` という２つの辞書を保持しており、プログラマはこれらにアクセスできます。

``data`` は入力データですが、これはバリデーションにより変更される場合があります。

``errors`` はエラーメッセージを保持しています。

.. code:: python

    data = {'first_name': ''}
    validator.validate(data)

    print(validator.data)
    # {}

    print(validator.errors)
    # {'first_name': 'This field is required'}

この例では ``first_name`` で１個のエラーが起こっています。
これは ``validate_not_empty()`` に渡す際、そのフィールドのデータを渡していない（空文字列を渡している）からです。
また ``data`` 辞書は空になっていますが、これはバリデータが値を渡していないからです。

すべてのバリデータにマッチするようなデータを渡した場合、 ``errors`` 辞書は空になり、
``data`` 辞書には値がセットされます:

.. code:: python

    data = {'first_name': 'Tim'}
    validator.validate(data)

    print(validator.data)
    # {'first_name': 'Tim'}

    print(validator.errors)
    # {}

``data`` 辞書には何らかのバリデータの後の値、強制変更された後の型、およびその他のカスタム修飾子がセットされます。
さらに、この同じバリデータインスタンスに対して別の新しい辞書データを渡すことで、バリデータを再利用できます。

データ型の強制変更
---------------------

データバリデーションの際に行われる最初のプロセスは、データ型の強制変更(coerce)です。

さまざまなフィールドがビルトインとして用意されています。完全なリストは API ドキュメントで確認してみてください。

フィールドの一例を示します。
これは単に、``IntegerField`` と同等の機能を実現する例を示すためのものです。

.. code:: python

    class CustomIntegerField(Field):
        def coerce(self, value):
            try:
                return int(value)
            except (TypeError, ValueError):
                raise ValidationError('coerce_int')

    class SimpleValidator(Validator):
        code = CustomIntegerField()

    validator = SimpleValidator()
    validator.validate({'code': 'text'})

    validator.data
    # {}

    validator.errors
    # {'code': 'Must be a valid integer.'}

利用可能なバリデータ
====================

``peewee_validates`` からインポートすることで利用可能になるビルトインのバリデータがたくさんあります。

* ``validate_email()`` - データがEメールアドレスであることをバリデート
* ``validate_equal(value)`` - データが ``value`` と等しいことをバリデート
* ``validate_function(method, **kwargs)`` - 第一引数がフィールド値、第二引数以降を ``kwargs`` として ``method`` を呼ぶことで、結果の正当性をバリデート
* ``validate_length(low, high, equal)`` - 長さが ``low`` と ``high`` の間もしくは ``equal`` と等しいかどうかをバリデート
* ``validate_none_of(values)`` - 値が ``values`` の中にないことをバリデート。``values`` には、呼ばれたら値を返すような callable も指定できます。
* ``validate_not_empty()`` - データが空でないことをバリデート
* ``validate_one_of(values)`` - 値が ``values`` の中にあることをバリデート。``values`` には、呼ばれたら値を返すような callable も指定できます。
* ``validate_range(low, high)`` - 値が ``low`` と ``high`` の間であることをバリデート
* ``validate_regexp(pattern, flags=0)`` - 値が ``patten`` にマッチすることをバリデート
* ``validate_required()`` - フィールドが存在することをバリデート

カスタムバリデータ
===================

フィールドバリデータは、 ``validator(field, data)`` シグニチャを持つ単なるメソッドです。
フィールドが ``Field`` インスタンス、 ``data`` が data 辞書として ``validate()`` に渡されます。

名前が常に "tim" であることを保証するためのバリデータを実装したい場合、たとえば以下のようになります:

.. code:: python

    def always_tim(field, data):
        if field.value and field.value != 'tim':
            raise ValidationError('not_tim')

    class SimpleValidator(Validator):
        name = StringField(validators=[always_tim])

    validator = SimpleValidator()
    validator.validate({'name': 'bob'})

    validator.errors
    # {'name': 'Validation failed.'}

これはあまり的確なエラーメッセージではありませんが、これをカスタマイズする方法は後でご紹介します。

さてここで、フィールドの長さをチェックするためのバリデータを実装することを考えてみましょう。長さは設定可能にしたいと思います。
やり方としては、パラメータを受け取って、バリデーション関数を返すようなバリデータを実装するというアプローチです。
基本的には、実際の validator 関数を別の関数でラップするようにします。たとえば以下のようになります:

.. code:: python

    def length(max_length):
        def validator(field, data):
            if field.value and len(field.value) > max_length:
                raise ValidationError('too_long')
        return validator

    class SimpleValidator(Validator):
        name = StringField(validators=[length(2)])

    validator = SimpleValidator()
    validator.validate({'name': 'bob'})

    validator.errors
    # {'name': 'Validation failed.'}

カスタムエラーメッセージ
=========================

これまでにお見せした例では、デフォルトのエラーメッセージが必ずしもわかりやすいものではありませんでした。
エラーメッセージは ``Meta`` クラスの ``messages`` 属性をセットすることで変更可能です。
エラーメッセージはキーで検索され、さらにオプションで頭にフィールド名を付加できます。

キーは、エラーが起こったときに ``ValidationError`` に渡された第一引数です。

.. code:: python

    class SimpleValidator(Validator):
        name = StringField(required=True)

        class Meta:
            messages = {
                'required': '値を入力してください.'
            }

これで、入力必須の項目のエラーメッセージはすべて "値を入力してください." になります。
さらに頭にフィールド名を付けることで、特定のフィールドのみ別のメッセージにすることができます。

.. code:: python

    class SimpleValidator(Validator):
        name = StringField(required=True)
        color = StringField(required=True)

        class Meta:
            messages = {
                'name.required': '名前を入力してください.',
                'required': '値を入力してください.',
            }

これで ``name`` フィールドのエラーメッセージは "名前を入力してください." に、
それ以外の必須フィールドについてはその他のエラーメッセージを使うようになります。

フィールドの除外／限定
=========================

バリデーションの際に特定のフィールドを限定または除外することができます。
これはクラスレベルか、もしくは ``validate()`` コール時に行います。

以下の例では ``validate()`` がコールされた時に、``name`` および ``color`` フィールドに限ってバリデートします:

.. code:: python

    class SimpleValidator(Validator):
        name = StringField(required=True)
        color = StringField(required=True)
        age = IntegerField(required=True)

        class Meta:
            only = ('name', 'color')

同様に、``validate()`` が呼ばれた際にオーバーライドも可能です:

.. code:: python

    validator = SimpleValidator()
    validator.validate(data, only=('color', 'name'))

これでクラスの定義は無視され、``color`` と ``name`` のみがバリデートされます。

バリデーションから特定のフィールドを除外する ``exclude`` 属性もあります。
使い方は ``only`` の書式と同様です。

モデルのバリデーション
=======================

ここまでの時点で Peewee に関することついては何も言及していないにも関わらず、
このパッケージがなぜ peewee-validates と呼ばれるのか、不思議に思われるかもしれません。
ここでその謎を解き明かします。このパッケージには ModelValidator クラスが含まれていますが、
これはすでに述べたように、モデルインスタンスをバリデートするのに使っています。

.. code:: python

    import peewee
    from peewee_validates import ModelValidator

    class Category(peewee.Model):
        code = peewee.IntegerField(unique=True)
        name = peewee.CharField(max_length=250)

    obj = Category(code=42)

    validator = ModelValidator(obj)
    validator.validate()

この例では、以下のように ModelValidator がバリデータをビルドしています:

.. code:: python

    unique_code_validator = validate_model_unique(
        Category.code, Category.select(), pk_field=Category.id, pk_value=obj.id)

    class CategoryValidator(Validator):
        code = peewee.IntegerField(
            required=True,
            validators=[unique_code_validator])
        name = peewee.StringField(required=True, max_length=250)

私たちのモデルの中で多くのものが定義され、自動的にバリデータの属性として変換されているのがわかります:

* name は必須の文字列
* name は 250 文字以下
* code は必須の整数
* code はテーブル内でユニークでなければならない

これでバリデータを使ってデータをバリデートできるようになりました。

デフォルトでは、これはモデルのインスタンスで直接データをバリデートしますが、
辞書を ``validates`` に渡すことでいつでもインスタンスの任意のデータをオーバーライドできます。

.. code:: python

    obj = Category(code=42)
    data = {'code': 'notnum'}

    validator = ModelValidator(obj)
    validator.validate(data)

    validator.errors
    # {'code': 'Must be a valid integer.'}

渡されたデータが数字ではないため、たとえインスタンスのデータが有効であったとしても、このバリデーションは失敗します。

``ModelValidator`` のサブクラスを作って、その中でこれまでに示したあらゆる要素を使うこともできます:

.. code:: python

    import peewee
    from peewee_validates import ModelValidator

    class CategoryValidator(ModelValidator):
        class Meta:
            messages = {
                'name.required': 'Enter your name.',
                'required': 'Please enter a value.',
            }

    validator = ModelValidator(obj)
    validator.validate(data)

ModelValidator についてバリデーションが成功したら、指定されたモデルインスタンスの中身は変更されます。

.. code:: python

    validator = ModelValidator(obj)

    obj.name
    # 'tim'

    validator.validate({'name': 'newname'})

    obj.name
    # 'newname'

フィールドのバリデーション
---------------------------

ModelValidator を使うと、標準の Validator クラスにはない機能が使えるようになります。

**ユニーク性**

Peewee のフィールドが ``unique=True`` で定義されている場合、そのフィールドにバリデータが追加され、
データベース内でそれがユニークであるかどうかが検査されます。これにより、
すでにデータベースに保存された値であっても、現在のインスタンスを除外するべきかどうかが判別できます。

**外部キー**

Peewee のフィールドが ``ForeignKeyField`` の場合、そのフィールドにバリデータが追加され、
データベースの関連するテーブルにその値があることが検査され、それが有効なインスタンスであることが保証されます。

**Many to Many**

Peewee のフィールドが  ``ManyToManyField`` の場合、そのフィールドにバリデータが追加され、
データベースの関連するテーブル（群）にその値があることが検査され、それが有効なインスタンスであることが保証されます。

**インデックスのバリデーション**

以下の例のように、モデルにユニークなインデックスを定義している場合、
（他のすべてのフィールドレベルのバリデーションが成功した後で）これについてもバリデートされます。

.. code:: python

    class Category(peewee.Model):
        code = peewee.IntegerField(unique=True)
        name = peewee.CharField(max_length=250)

        class Meta:
            indexes = (
                (('name', 'code'), True),
            )

フィールドのオーバーライド
==========================

モデルのフィールドのバリデート方法を変更する必要がある場合、
単にカスタムクラス内でそのフィールドをオーバーライドするだけで済みます。
以下のモデルについて例を示します:

.. code:: python

    class Category(peewee.Model):
        code = peewee.IntegerField(required=True)

これにより、 required バリデータを持つ ``code`` に対応するフィールドが生成されます。

.. code:: python

    class CategoryValidator(ModelValidator):
        code = IntegerField(required=False)

    validator = CategoryValidator(category)
    validator.validate()

これで ``validate`` への呼び出しが起こっても、``code`` は必須ではなくなります。

バリデート後の振舞いをオーバーライドする
========================================

クリーニング
-------------

``validate()`` の中でフィールドレベルのデータがすべてバリデートされると、
その結果データは上位に返される前に ``clean()`` メソッドに渡されます。
このメソッドをオーバーライドすることで、好きなバリデーションを実行したり、
また返すデータを変更したりすることが可能です。

.. code:: python

    class MyValidator(Validator):
        name1 = StringField()
        name2 = StringField()

        def clean(self, data):
            # name1がname2と等しいことを保証する
            if data['name1'] != data['name2']:
                raise ValidationError('name_different')
            # そしてこれらが同じ場合、大文字に変更する
            data['name1'] = data['name1'].upper()
            data['name2'] = data['name2'].upper()
            return data

        class Meta:
            messages = {
                'name_different': '名前は同じでなければなりません.'
            }

フィールドを動的に追加する
-----------------------------

必要であれば、バリデータインスタンスに対して動的にフィールドを追加することが可能です。
追加されたフィールドは ``_meta.fields`` 辞書に格納され、その後これらを自由に操作できます。

.. code:: python

    validator = MyValidator()
    validator._meta.fields['newfield'] = IntegerField(required=True)
