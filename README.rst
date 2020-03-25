peewee-validates
################

`Peewee ORM <http://docs.peewee-orm.com/>`_ のための、シンプルかつフレキシブルなモデルとデータバリデータ。

.. image:: https://img.shields.io/travis/timster/peewee-validates.svg?style=flat
    :target: https://travis-ci.org/timster/peewee-validates
    :alt: Build Status

.. image:: https://img.shields.io/coveralls/timster/peewee-validates.svg?style=flat
    :target: https://coveralls.io/r/timster/peewee-validates
    :alt: Code Coverage

.. image:: https://img.shields.io/pypi/v/peewee-validates.svg?style=flat
    :target: https://pypi.python.org/pypi/peewee-validates
    :alt: Version

.. image:: https://img.shields.io/pypi/dm/peewee-validates.svg?style=flat
    :target: https://pypi.python.org/pypi/peewee-validates
    :alt: Downloads

.. image:: https://readthedocs.org/projects/peewee-validates/badge/?version=latest
    :target: https://peewee-validates.readthedocs.io
    :alt: Documentation

必要事項
============

* python >= 3.3
* peewee >= 2.8.2 (including Peewee 3)
* python-dateutil >= 2.5.0

インストール
==============

このパッケージは pip でインストールできます:

::

    pip install peewee-validates

使い方
========

peewee-validates でできることを以下にチラ見せします:

.. code:: python

    import peewee
    from peewee_validates import ModelValidator

    class Category(peewee.Model):
        code = peewee.IntegerField(unique=True)
        name = peewee.CharField(null=False, max_length=250)

    obj = Category(code=42)

    validator = ModelValidator(obj)
    validator.validate()

    print(validator.errors)

    # {'name': 'This field is required.', 'code': 'Must be a unique value.'}

実際のところ、モデルさえ必要としない汎用的なバリデータもあります:

.. code:: python

    from peewee_validates import Validator, StringField

    class SimpleValidator(Validator):
        name = StringField(required=True, max_length=250)
        code = StringField(required=True, max_length=4)

    validator = SimpleValidator(obj)
    validator.validate({'code': 'toolong'})

    print(validator.errors)

    # {'name': 'This field is required.', 'code': 'Must be at most 5 characters.'}

ドキュメント
=============

詳細は `完全なドキュメント <http://peewee-validates.readthedocs.io>`_ を参照してください。

日本語ドキュメントは `https://net-newbie.com/peewee-validates/ <https://net-newbie.com/peewee-validates/>`_ にあります。
