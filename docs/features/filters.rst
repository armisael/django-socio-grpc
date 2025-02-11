.. _filters:

Filters
==========

**Filters** are used to filter the *queryset* of your service. You can use built-in filters from `django filters <https://www.django-rest-framework.org/api-guide/filtering/>`_ or create your own filters.


Description
-----------

This page will explain how to set up filters in your app services. Filter behave the same as `DRF filters <https://www.django-rest-framework.org/api-guide/filtering/>`_.

We will reproduce an DRF example and demonstrate how to use `django filters <https://www.django-rest-framework.org/api-guide/filtering/>`_ in DSG.


Filtering against the current user (context.user)
--------------------------------------------------

You might want to filter the *queryset* to ensure that only results relevant to the currently authenticated user making the request are returned.

You can do so by filtering based on the value of context.user.

For example:

.. code-block:: python

    #quickstart/services.py
    from django_socio_grpc import generics

    from quickstart.models import Post
    from quickstart.serializer import PostProtoSerializer

    # This service will have all the CRUD actions
    class PostService(generics.AsyncModelService):
        queryset = Post.objects.all()
        serializer_class = PostProtoSerializer

        def get_queryset(self):
            """
            This view should return a list of all the posts
            for the currently authenticated user.
            """
            user = self.context.user
            return Post.objects.filter(user=user)


Filtering against request fields (request.user)
------------------------------------------------

Another style of filtering might involve restricting the *queryset* based on some fields of the request.

For example with a message like:

.. code-block:: proto

    message PostListRequest {
        string user = 1; // user here is the id/uuid of the user
    }


.. code-block:: python

    #quickstart/services.py
    from django_socio_grpc import generics

    from quickstart.models import Post
    from quickstart.serializer import PostProtoSerializer

    # This service will have all the CRUD actions
    class PostService(generics.AsyncModelService):
        queryset = Post.objects.all()
        serializer_class = PostProtoSerializer

        def get_queryset(self):
            """
            This view should return a list of all the posts
            for the currently authenticated user.
            """
            user = self.request.user
            return Post.objects.filter(user=user)


Filtering against metadata
---------------------------

A final example of filtering the initial queryset would be to use the `grpc metadata <https://github.com/grpc/grpc/tree/master/examples/python/metadata>`_

In DSG, metadata is used to replace the query parameters systems. It is very flexible as it's not specified through the proto file.

The main inconveniences are:
    * The metadata are not binary serialized so passing a lot of data as filters may result in **poor performance**
    * They not exported in the proto so **not documented** by default.

.. note::
    We are currently looking for filtering best practices. See https://github.com/socotecio/django-socio-grpc/issues/247.


.. code-block:: python

    # server
    # quickstart/services.py
    from django_socio_grpc import generics

    from quickstart.models import Post
    from quickstart.serializer import PostProtoSerializer

    # This service will have all the CRUD actions
    class PostService(generics.AsyncModelService):
        queryset = Post.objects.all()
        serializer_class = PostProtoSerializer

        def get_queryset(self):
            """
            This view should return a list of all the posts
            for the currently authenticated user.
            """
            user = self.context.grpc_request_metadata["FILTERS"]["user"]
            # Next line also working to make REST library working
            # user = self.context.query_params["user"]
            return Post.objects.filter(user=user)

    # client
    import asyncio
    import grpc
    import json

    async def main():
        async with grpc.aio.insecure_channel("localhost:50051") as channel:
            quickstart_client = quickstart_pb2_grpc.PostControllerStub(channel)

            request = quickstart_pb2.PostListRequest()
            filter_as_dict = {"user": "be76adbb-73c3-4d65-b823-66b3276df38b"}
            # json.dumps is used to serialize the dict in the right string format for improved syntax checking
            metadata = (("filters", (json.dumps(filter_as_dict))),)

            response = await quickstart_client.List(request, metadata=metadata)

    if __name__ == "__main__":
        asyncio.run(main())


DjangoFilterBackend
-------------------

First install `django_filters <https://django-filter.readthedocs.io/en/stable/guide/install.html>`_.
You can also read their `doc for the DRF integration if you are not familiar with it <https://django-filter.readthedocs.io/en/stable/guide/rest_framework.html>`_.

============================
Register DjangoFilterBackend
============================

You can see a fully working example `in DSG example repo <https://github.com/socotecio/django-socio-grpc-example/blob/main/backend/example_bib_app/services.py>`_.

You can register it **by service** or **globally**:

* Register DjangoFilterBackend **by service**:

.. code-block:: python

    # quickstart/services.py
    from django_socio_grpc import generics
    from django_filters.rest_framework import DjangoFilterBackend

    class PostService(generics.AsyncModelService):
        ...
        filter_backends = [DjangoFilterBackend]


* Register DjangoFilterBackend **globally**. :ref:`See DSG settings DEFAULT_FILTER_BACKENDS<default_filter_backends_settings>`:

.. code-block:: python

    # settings.py
    GRPC_FRAMEWORK = {
        ...
        "DEFAULT_FILTER_BACKENDS": [DjangoFilterBackend],
        ...
    }


======================
Declare filter fields
======================

There is two way to defining filter fields.

* Using ``filterset_fields`` service attribute

.. code-block:: python

    # server
    # quickstart/services.py
    from django_socio_grpc import generics
    from django_filters.rest_framework import DjangoFilterBackend
    from quickstart.models import Post
    from quickstart.serializer import PostProtoSerializer

    # This service will have all the CRUD actions
    class PostService(generics.AsyncModelService):
        queryset = Post.objects.all()
        serializer_class = PostProtoSerializer
        filter_backends = [DjangoFilterBackend]
        filterset_fields = ['user']


* Using ``filterset_class`` service attribute. See `here for more details <https://django-filter.readthedocs.io/en/stable/ref/filterset.html>`_

.. code-block:: python

    # server
    # quickstart/services.py
    from django_socio_grpc import generics
    from django_filters.rest_framework import DjangoFilterBackend
    from quickstart.models import Post
    from quickstart.serializer import PostProtoSerializer
    from django_filters import rest_framework as filters

    class PostFilter(filters.FilterSet):
        user = filters.UUIDFilter(field_name="user")

        class Meta:
            model = Post
            fields = ['user']

    # This service will have all the CRUD actions
    class PostService(generics.AsyncModelService):
        queryset = Post.objects.all()
        serializer_class = PostProtoSerializer
        filter_backends = [DjangoFilterBackend]
        filterset_class = PostFilter


========
Using it
========

We use grpc metadata to make the filters works out of the box.

For more example you can see the `client in DSG example repo <https://github.com/socotecio/django-socio-grpc-example/blob/main/backend/bib_example_filter_client.py>`_

.. code-block:: python

    # client
    import asyncio
    import grpc
    import json

    async def main():
        async with grpc.aio.insecure_channel("localhost:50051") as channel:
            quickstart_client = quickstart_pb2_grpc.PostControllerStub(channel)

            request = quickstart_pb2.PostListRequest()
            # filters only the user with id "be76adbb-73c3-4d65-b823-66b3276df38b"
            filter_as_dict = {"user": "be76adbb-73c3-4d65-b823-66b3276df38b"}
            metadata = (("filters", (json.dumps(filter_as_dict))),)

            response = await quickstart_client.List(request, metadata=metadata)

    if __name__ == "__main__":
        asyncio.run(main())


For web usage see `How to web: Using js client<_using_js_client>_`


SearchFilter
-------------

DSG also supports the `DRF SearchFilter <https://www.django-rest-framework.org/api-guide/filtering/#searchfilter>`_

Refer to the DRF docs for implementation details and specific lookup.

.. code-block:: python

    # server
    # quickstart/services.py
    from django_socio_grpc import generics
    from rest_framework import filters
    from quickstart.models import Post
    from quickstart.serializer import PostProtoSerializer

    # This service will have all the CRUD actions
    class PostService(generics.AsyncModelService):
        queryset = Post.objects.all()
        serializer_class = PostProtoSerializer
        filter_backends = [filters.SearchFilter]
        search_fields = ['user__full_name']

    # client
    import asyncio
    import grpc
    import json

    async def main():
        async with grpc.aio.insecure_channel("localhost:50051") as channel:
            quickstart_client = quickstart_pb2_grpc.PostControllerStub(channel)

            request = quickstart_pb2.PostListRequest()
            filter_as_dict = {"search": "test-user"}  # search for "test-user" in user__full_name
            metadata = (("filters", (json.dumps(filter_as_dict))),)

            response = await quickstart_client.List(request, metadata=metadata)

    if __name__ == "__main__":
        asyncio.run(main())


OrderingFilter
--------------

OrderingFilters are used to control the ordering of the results.

DSG also support the `DRF OrderingFilter <https://www.django-rest-framework.org/api-guide/filtering/#orderingfilter>`_

Refer to the `DRF doc <https://www.django-rest-framework.org/api-guide/filtering/#orderingfilter>`_ for implementation details and specific lookup.

.. code-block:: python

    # server
    # quickstart/services.py
    from django_socio_grpc import generics
    from rest_framework import filters
    from quickstart.models import Post
    from quickstart.serializer import PostProtoSerializer

    # This service will have all the CRUD actions
    class PostService(generics.AsyncModelService):
        queryset = Post.objects.all()
        serializer_class = PostProtoSerializer
        filter_backends = [filters.OrderingFilter]
        ordering_fields = ['pub_date']

    # client
    import asyncio
    import grpc
    import json

    async def main():
        async with grpc.aio.insecure_channel("localhost:50051") as channel:
            quickstart_client = quickstart_pb2_grpc.PostControllerStub(channel)

            request = quickstart_pb2.PostListRequest()
            # order by descending pub_date 
            filter_as_dict = {"ordering": "-pub_date"}
            metadata = (("filters", (json.dumps(filter_as_dict))),)

            response = await quickstart_client.List(request, metadata=metadata)

    if __name__ == "__main__":
        asyncio.run(main())
