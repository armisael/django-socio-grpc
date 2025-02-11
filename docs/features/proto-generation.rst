.. _proto-generation:

Proto generation
================

DSG provides an automatic way to generate proto files from your django models and serivces with the  :ref:`generateproto command <commands-generate-proto>`.

To be able to generate proto files you need to register your service first.
To do so please refer to:

* :ref:`Getting started: Register Services <quickstart-register-services>` for quick registration
* :ref:`Services Registry <services-registry>` for more understanding

Description
-----------

**Protobuf** is a serialization format developed by Google and used in gRPC. It is a binary format that is optimized to be sent over the network. It is also used to generate code for the client and server side.

This format is developed in a large community in its own project. The following documentation expects that you are at least familiar with the concepts of **Protobuf**. 
If you are not, please read `their documentation <https://protobuf.dev/getting-started/pythontutorial/>_` first.

Proto files contain the classes, descriptors and controller logic (``pb2.py`` files) and proto message syntax (``.proto`` file) necessary to run a grpc server.

In DSG, proto files are generated from a ``grpc_action`` request / response contents (see :ref:`grpc action <grpc_action>` for more use cases).

To simplify usage ``grpc_action`` are automatically generated from the :ref:`Proto Serializer <proto-serializers>` when using :ref:`Generic Mixins <Generic Mixins>`.

In order to generate these files and its contents, there is a :ref:`django command <commands>` to run whenever you add a ``grpc_action``, a Service or modify your request / response:

Usage
-----
.. code-block:: bash

    python manage.py generateproto


.. list-table:: Options Available:
    :widths: 15 10 30 45
    :header-rows: 1

    * - Option
      - Shortcut
      - Default value
      - Description
    * - --project
      - -p
      - Use DJANGO_SETTINGS_MODULE first folder
      - Name of the django project that is use in the proto package name.
    * - --dry-run
      - -dr
      - False
      - Print in terminal the protofile content without writing it to a file or generate new python code.
    * - --no-generate-pb2
      - -nopb2
      - False
      - Avoid generating python file. Only proto.
    * - --check
      - -c
      - False
      - Check if the current protofile is the same that one that will be generated by a new command to be sur your api is sync with your models.
    * - --custom-verbose
      - -cv
      - 0
      - Number from 1 to 4 indicating the verbose level of the generation.
    * - --directory
      - -d
      - None
      - Directory where the proto files will be generated. Default will be in the apps directories



Example
-------

.. code-block:: python

    # quickstart/models.py
    from django.db import models


    class User(models.Model):
        full_name = models.CharField(max_length=70)

        def __str__(self):
            return self.full_name

    # quickstart/serializers.py
    from django_socio_grpc import proto_serializers
    from rest_framework import serializers
    from quickstart.models import User, Post, Comment


    class UserProtoSerializer(proto_serializers.ModelProtoSerializer):
        # This line is written here as an example,
        # but can be removed as the serializer integrates all the fields in the model
        full_name = serializers.CharField(allow_blank=True)
        class Meta:
            model = User
            fields = "__all__"

    # Service
    from django_socio_grpc import generics
    from django_socio_grpc.decorators import grpc_action
    from ..models import User
    from ..serializers import UserProtoSerializer

    # inherits from AsyncModelService, therefore will register all default CRUD actions.
    class UserService(generics.AsyncModelService):
        queryset = User.objects.all()
        serializer_class = UserProtoSerializer

        @grpc_action
        async def SomeCustomMethod(
            request=[{"name": "foo", "type": "string"}],
            response=[{"name": "bar", "type": "string"}],
            response_stream=True
        ):
            # logic here
            pass

    # quickstart/handlers.py
    from django_socio_grpc.services.app_handler_registry import AppHandlerRegistry
    from quickstart.services import UserService

    def grpc_handlers(server):
        app_registry = AppHandlerRegistry("quickstart", server)
        app_registry.register(UserService)

At the root of your project, run:

.. code-block:: bash

    python manage.py generateproto

If command executed successfully, you will see inside your user app, a grpc folder with two .py files, (``user_pb2.py`` and ``user_pb2_grpc.py``)
and a ``user.proto`` file. ``user.proto`` file should contain these lines:

.. code-block:: proto

    syntax = "proto3";

    package doc_example.generate_proto_doc;

    import "google/protobuf/empty.proto";

    service UserController {
        rpc List(UserListRequest) returns (UserListResponse) {}
        rpc Create(UserRequest) returns (UserResponse) {}
        rpc Retrieve(UserRetrieveRequest) returns (UserResponse) {}
        rpc Update(UserRequest) returns (UserResponse) {}
        rpc Destroy(UserDestroyRequest) returns (google.protobuf.Empty) {}
        rpc SomeCustomMethod(SomeCustomMethodRequest) returns (stream SomeCustomMethodResponse) {}
    }

    message UserResponse {
        string id = 1;
        string full_name = 2;
    }

    message UserListRequest {
    }

    message UserListResponse {
        repeated UserResponse results = 1;
    }

    message UserRequest {
        string id = 1;
        string full_name = 2;
    }

    message UserRetrieveRequest {
        string id = 1;
    }

    message UserDestroyRequest {
        string id = 1;
    }

    message SomeCustomMethodRequest {
        string foo = 1;
    }

    message SomeCustomMethodResponse {
        string bar = 1;
    }


Note: these files are meant to be read only, please do not modify, since they might be overwritten by a next generation call. 
You can use the .proto file as a reference to verify whether
or not your serializer fields were correctly mapped but you should not try to modify them manually.

For more example and use case go to :ref:`Generic Mixins <Generic Mixins>` and :ref:`grpc action <grpc_action>`



Field number attribution
-------------------------

COMING SOON
