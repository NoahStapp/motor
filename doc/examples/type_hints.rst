
.. _type_hints-example:

Type Hints
==========

.. warning:: As of May 14th, 2025, Motor is deprecated in favor of the GA release of the PyMongo Async API.
  No new features will be added to Motor, and only bug fixes will be provided until it reaches end of life on May 14th, 2026.
  After that, only critical bug fixes will be made until final support ends on May 14th, 2027.
  We strongly recommend migrating to the PyMongo Async API while Motor is still supported.
  For help transitioning, see the `Migrate to PyMongo Async guide <https://www.mongodb.com/docs/languages/python/pymongo-driver/current/reference/migration/>`_.


As of version 3.3.0, Motor ships with `type hints`_. With type hints, Python
type checkers can easily find bugs before they reveal themselves in your code.

If your IDE is configured to use type hints,
it can suggest more appropriate completions and highlight errors in your code.
Some examples include `PyCharm`_,  `Sublime Text`_, and `Visual Studio Code`_.

You can also use the `mypy`_ tool from your command line or in Continuous Integration tests.

All of the public APIs in Motor are fully type hinted, and
several of them support generic parameters for the
type of document object returned when decoding BSON documents.

Due to `limitations in mypy`_, the default
values for generic document types are not yet provided (they will eventually be ``Dict[str, any]``).

For a larger set of examples that use types, see the Motor `test_typing module`_.

If you would like to opt out of using the provided types, add the following to
your `mypy config`_: ::

    [mypy-motor]
    follow_imports = False


Basic Usage
-----------

Note that a type for :class:`~motor.motor_asyncio.AsyncIOMotorClient` must be specified.  Here we use the
default, unspecified document type:

.. code-block:: python

   from motor.motor_asyncio import AsyncIOMotorClient


   async def main():
       client: AsyncIOMotorClient = AsyncIOMotorClient()
       collection = client.test.test
       inserted = await collection.insert_one({"x": 1, "tags": ["dog", "cat"]})
       retrieved = await collection.find_one({"x": 1})
       assert isinstance(retrieved, dict)

For a more accurate typing for document type you can use:

.. code-block:: python

   from typing import Any, Dict
   from motor.motor_asyncio import AsyncIOMotorClient


   async def main():
       client: AsyncIOMotorClient[Dict[str, Any]] = AsyncIOMotorClient()
       collection = client.test.test
       inserted = await collection.insert_one({"x": 1, "tags": ["dog", "cat"]})
       retrieved = await collection.find_one({"x": 1})
       assert isinstance(retrieved, dict)

Typed Client
------------

:class:`~motor.motor_asyncio.AsyncIOMotorClient` is generic on the document type used to decode BSON documents.

You can specify a :class:`~bson.raw_bson.RawBSONDocument` document type:

.. code-block:: python

   from motor.motor_asyncio import AsyncIOMotorClient
   from bson.raw_bson import RawBSONDocument


   async def main():
       client = AsyncIOMotorClient(document_class=RawBSONDocument)
       collection = client.test.test
       inserted = await collection.insert_one({"x": 1, "tags": ["dog", "cat"]})
       result = await collection.find_one({"x": 1})
       assert isinstance(result, RawBSONDocument)

Subclasses of :py:class:`collections.abc.Mapping` can also be used, such as :class:`~bson.son.SON`:

.. code-block:: python

   from bson import SON
   from motor.motor_asyncio import AsyncIOMotorClient


   async def main():
       client = AsyncIOMotorClient(document_class=SON[str, int])
       collection = client.test.test
       inserted = await collection.insert_one({"x": 1, "y": 2})
       result = await collection.find_one({"x": 1})
       assert result is not None
       assert result["x"] == 1

Note that when using :class:`~bson.son.SON`, the key and value types must be given, e.g. ``SON[str, Any]``.


Typed Collection
----------------

You can use :py:class:`~typing.TypedDict` when using a well-defined schema for the data in a
:class:`~motor.motor_asyncio.AsyncIOMotorClient`. Note that all `schema validation`_ for inserts and updates is done on the server.
These methods automatically add an "_id" field.

.. code-block:: python

   from typing import TypedDict
   from motor.motor_asyncio import AsyncIOMotorClient
   from motor.motor_asyncio import AsyncIOMotorCollection


   class Movie(TypedDict):
       name: str
       year: int


   async def main():
       client: AsyncIOMotorClient = AsyncIOMotorClient()
       collection: AsyncIOMotorCollection[Movie] = client.test.test
       inserted = await collection.insert_one(Movie(name="Jurassic Park", year=1993))
       result = await collection.find_one({"name": "Jurassic Park"})
       assert result is not None
       assert result["year"] == 1993
       # This will raise a type-checking error, despite being present, because it is added by Motor.
       assert result["_id"]  # type:ignore[typeddict-item]

This same typing scheme works for all of the insert methods (:meth:`~motor.motor_asyncio.AsyncIOMotorCollection.insert_one`,
:meth:`~motor.motor_asyncio.AsyncIOMotorCollection.insert_many`, and :meth:`~motor.motor_asyncio.AsyncIOMotorCollection.bulk_write`).
For ``bulk_write`` both :class:`~pymongo.operations.InsertOne` and :class:`~pymongo.operations.ReplaceOne` operators are generic.

.. code-block:: python

   from typing import TypedDict
   from motor.motor_asyncio import AsyncIOMotorClient
   from motor.motor_asyncio import AsyncIOMotorCollection
   from pymongo.operations import InsertOne


   async def main():
       client: AsyncIOMotorClient = AsyncIOMotorClient()
       collection: AsyncIOMotorCollection[Movie] = client.test.test
       inserted = await collection.bulk_write(
           [InsertOne(Movie(name="Jurassic Park", year=1993))]
       )
       result = await collection.find_one({"name": "Jurassic Park"})
       assert result is not None
       assert result["year"] == 1993
       # This will raise a type-checking error, despite being present, because it is added by Motor.
       assert result["_id"]  # type:ignore[typeddict-item]

Modeling Document Types with TypedDict
--------------------------------------

You can use :py:class:`~typing.TypedDict` to model structured data.
As noted above, Motor will automatically add an ``_id`` field if it is not present. This also applies to TypedDict.
There are three approaches to this:

1. Do not specify ``_id`` at all. It will be inserted automatically, and can be retrieved at run-time, but will yield a type-checking error unless explicitly ignored.

2. Specify ``_id`` explicitly. This will mean that every instance of your custom TypedDict class will have to pass a value for ``_id``.

3. Make use of :py:class:`~typing.NotRequired`. This has the flexibility of option 1, but with the ability to access the ``_id`` field without causing a type-checking error.

Note: to use :py:class:`~typing.NotRequired` in earlier versions of Python (<3.11), use the ``typing_extensions`` package.

.. code-block:: python

   from typing import TypedDict, NotRequired
   from motor.motor_asyncio import AsyncIOMotorClient
   from motor.motor_asyncio import AsyncIOMotorCollection
   from bson import ObjectId


   class Movie(TypedDict):
       name: str
       year: int


   class ExplicitMovie(TypedDict):
       _id: ObjectId
       name: str
       year: int


   class NotRequiredMovie(TypedDict):
       _id: NotRequired[ObjectId]
       name: str
       year: int


   async def main():
       client: AsyncIOMotorClient = AsyncIOMotorClient()
       collection: AsyncIOMotorCollection[Movie] = client.test.test
       inserted = await collection.insert_one(Movie(name="Jurassic Park", year=1993))
       result = await collection.find_one({"name": "Jurassic Park"})
       assert result is not None
       # This will yield a type-checking error, despite being present, because it is added by Motor.
       assert result["_id"]  # type:ignore[typeddict-item]

       collection: AsyncIOMotorCollection[ExplicitMovie] = client.test.test
       # Note that the _id keyword argument must be supplied
       inserted = await collection.insert_one(
           ExplicitMovie(_id=ObjectId(), name="Jurassic Park", year=1993)
       )
       result = await collection.find_one({"name": "Jurassic Park"})
       assert result is not None
       # This will not raise a type-checking error.
       assert result["_id"]

       collection: AsyncIOMotorCollection[NotRequiredMovie] = client.test.test
       # Note the lack of _id, similar to the first example
       inserted = await collection.insert_one(
           NotRequiredMovie(name="Jurassic Park", year=1993)
       )
       result = await collection.find_one({"name": "Jurassic Park"})
       assert result is not None
       # This will not raise a type-checking error, despite not being provided explicitly.
       assert result["_id"]


Typed Database
--------------

While less common, you could specify that the documents in an entire database
match a well-defined schema using :py:class:`~typing.TypedDict`.

.. code-block:: python

   from typing import TypedDict
   from motor.motor_asyncio import AsyncIOMotorClient
   from motor.motor_asyncio import AsyncIOMotorDatabase


   class Movie(TypedDict):
       name: str
       year: int


   async def main():
       client: AsyncIOMotorClient = AsyncIOMotorClient()
       db: AsyncIOMotorDatabase[Movie] = client.test
       collection = db.test
       inserted = await collection.insert_one({"name": "Jurassic Park", "year": 1993})
       result = await collection.find_one({"name": "Jurassic Park"})
       assert result is not None
       assert result["year"] == 1993

Typed Command
-------------
When using the :meth:`~motor.motor_asyncio.AsyncIOMotorDatabase.command`, you can specify the document type by providing a custom :class:`~bson.codec_options.CodecOptions`:

.. code-block:: python

   from motor.motor_asyncio import AsyncIOMotorClient
   from bson.raw_bson import RawBSONDocument
   from bson import CodecOptions


   async def main():
       client: AsyncIOMotorClient = AsyncIOMotorClient()
       options = CodecOptions(RawBSONDocument)
       result = await client.admin.command("ping", codec_options=options)
       assert isinstance(result, RawBSONDocument)

Custom :py:class:`collections.abc.Mapping` subclasses and :py:class:`~typing.TypedDict` are also supported.
For :py:class:`~typing.TypedDict`, use the form: ``options: CodecOptions[MyTypedDict] = CodecOptions(...)``.

Typed BSON Decoding
-------------------
You can specify the document type returned by :mod:`bson` decoding functions by providing :class:`~bson.codec_options.CodecOptions`:

.. code-block:: python

      from typing import Any, Dict
      from bson import CodecOptions, encode, decode


      class MyDict(Dict[str, Any]):
          pass


      def foo(self):
          return "bar"


      options = CodecOptions(document_class=MyDict)
      doc = {"x": 1, "y": 2}
      bsonbytes = encode(doc, codec_options=options)
      rt_document = decode(bsonbytes, codec_options=options)
      assert rt_document.foo() == "bar"

:class:`~bson.raw_bson.RawBSONDocument` and :py:class:`~typing.TypedDict` are also supported.
For :py:class:`~typing.TypedDict`, use  the form: ``options: CodecOptions[MyTypedDict] = CodecOptions(...)``.


Troubleshooting
---------------

Client Type Annotation
~~~~~~~~~~~~~~~~~~~~~~
If you forget to add a type annotation for a :class:`~motor.motor_asyncio.AsyncIOMotorClient` object you may get the following ``mypy`` error:

.. code-block:: python

  from motor.motor_asyncio import AsyncIOMotorClient

  client = AsyncIOMotorClient()  # error: Need type annotation for "client"

The solution is to annotate the type as ``client: AsyncIOMotorClient`` or ``client: AsyncIOMotorClient[Dict[str, Any]]``.  See `Basic Usage`_.

Incompatible Types
~~~~~~~~~~~~~~~~~~
If you use the generic form of :class:`~motor.motor_asyncio.AsyncIOMotorClient` you
may encounter a ``mypy`` error like:

.. code-block:: python

   from motor.motor_asyncio import AsyncIOMotorClient


   async def main():
       client: AsyncIOMotorClient = AsyncIOMotorClient()
       await client.test.test.insert_many(
           {"a": 1}
       )  # error: Dict entry 0 has incompatible type "str": "int";
       # expected "Mapping[str, Any]": "int"


The solution is to use ``client: AsyncIOMotorClient[Dict[str, Any]]`` as used in
`Basic Usage`_ .

Actual Type Errors
~~~~~~~~~~~~~~~~~~

Other times ``mypy`` will catch an actual error, like the following code:

.. code-block:: python

   from motor.motor_asyncio import AsyncIOMotorClient
   from typing import Mapping


   async def main():
       client: AsyncIOMotorClient = AsyncIOMotorClient()
       await client.test.test.insert_one(
           [{}]
       )  # error: Argument 1 to "insert_one" of "Collection" has
       # incompatible type "List[Dict[<nothing>, <nothing>]]";
       # expected "Mapping[str, Any]"

In this case the solution is to use ``insert_one({})``, passing a document instead of a list.

Another example is trying to set a value on a :class:`~bson.raw_bson.RawBSONDocument`, which is read-only.:

.. code-block:: python

   from bson.raw_bson import RawBSONDocument
   from motor.motor_asyncio import AsyncIOMotorClient


   async def main():
       client = AsyncIOMotorClient(document_class=RawBSONDocument)
       coll = client.test.test
       doc = {"my": "doc"}
       await coll.insert_one(doc)
       retrieved = await coll.find_one({"_id": doc["_id"]})
       assert retrieved is not None
       assert len(retrieved.raw) > 0
       retrieved["foo"] = "bar"  # error: Unsupported target for indexed assignment
       # ("RawBSONDocument")  [index]

.. _PyCharm: https://www.jetbrains.com/help/pycharm/type-hinting-in-product.html
.. _Visual Studio Code: https://code.visualstudio.com/docs/languages/python
.. _Sublime Text: https://github.com/sublimelsp/LSP-pyright
.. _type hints: https://docs.python.org/3/library/typing.html
.. _mypy: https://mypy.readthedocs.io/en/stable/cheat_sheet_py3.html
.. _limitations in mypy: https://github.com/python/mypy/issues/3737
.. _mypy config: https://mypy.readthedocs.io/en/stable/config_file.html
.. _test_typing module: https://github.com/mongodb/motor/blob/master/test/test_typing.py
.. _schema validation: https://www.mongodb.com/docs/manual/core/schema-validation/#when-to-use-schema-validation
