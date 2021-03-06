.. _extending:
.. module:: marshmallow

Extending Serializers
=====================

Handling Errors
---------------

By default, :meth:`Schema.dump` and :meth:`Schema.load` will return validation errors as a dictionary (unless ``strict`` mode is enabled).

You can register a custom error-handling function for a :class:`Schema` using the :meth:`Schema.error_handler` decorator. The function receives the serializer instance, the errors dictionary, and the original object to be serialized.


.. code-block:: python

    import logging

    class AppError(Exception):
        pass

    class UserSchema(Schema):
        email = fields.Email()

    @UserSchema.error_handler
    def handle_errors(serializer, errors, obj):
        logging.error(errors)
        raise AppError('An error occurred while serializing {0}'.format(obj))

    invalid = User('Foo Bar', email='invalid-email')
    serializer = UserSchema()
    serializer.dump(invalid)  # raises AppError
    serializer.load({'email': 'invalid-email'})  # raises AppError



Transforming Data
-----------------

The :func:`Schema.data_handler <Schema.data_handler>` decorator can be used to register data post-processing functions that transform the serialized data. The function receives the serializer instance, the serialized data dictionary, and the original object to be serialized. It should return the transformed data.

One use case might be to add a "root" namespace for a serialized object.

.. code-block:: python

    class UserSchema(Schema):
        NAME = 'user'
        name = fields.String()
        email = fields.Email()

    @UserSchema.data_handler
    def add_root(serializer, data, obj):
        return {
            serializer.NAME: data
        }

    user = User('Monty Python', email='monty@python.org')
    UserSchema(user).data
    # {
    #     'user': {
    #         'name': 'Monty Python',
    #         'email': 'monty@python.org'
    #     }
    # }

.. note::

    It is possible to register multiple data handlers for a single serializer.

Error Handlers and Data Handlers as Class Members
-------------------------------------------------

You can register error handlers and data handlers as class members. This might be useful if when defining an abstract serializer class.

.. code-block:: python

    class BaseSchema(Schema):
        """A customized serializer with error handling and post-processing behavior."""
        __error_handler__ = handle_errors  # A function
        __data_handlers__ = [add_root]  # A list of functions



Extending "class Meta" Options
--------------------------------

``class Meta`` options are a way to configure and modify a :class:`Schema's <Schema>` behavior. See the :class:`API docs <Schema>` for a listing of available options.

You can add custom ``class Meta`` options by subclassing :class:`SchemaOpts`.

Example: Adding a Namespace to Serialized Output
++++++++++++++++++++++++++++++++++++++++++++++++

Let's build upon the example above for adding a root namespace to serialized output. This time, we will create a custom base serializer with additional ``class Meta`` options.

::

    # Example outputs
    {
        'user': {
            'name': 'Keith',
            'email': 'keith@stones.com'
        }
    }
    # List output
    {
        'users': [{'name': 'Keith'}, {'name': 'Mick'}]
    }


First, we'll add our namespace configuration to a custom options class.

.. code-block:: python

    from marshmallow import Schema, SchemaOpts

    class NamespaceOpts(SchemaOpts):
        """Same as the default class Meta options, but adds "name" and
        "plural_name" options for namespacing.
        """

        def __init__(self, meta):
            SchemaOpts.__init__(self, meta)
            self.name = getattr(meta, 'name', None)
            self.plural_name = getattr(meta, 'plural_name', self.name)


Then we create a custom :class:`Schema` that uses our options class.

.. code-block:: python


    class NamespacedSchema(Schema):
        OPTIONS_CLASS = NamespaceOpts

        def _postprocess(self, data, obj):
            """Execute any postprocessing steps, including adding a namespace to the final
            output.
            """
            data = Schema._postprocess(self, data)
            if self.opts.name:   # Add namespace
                namespace = self.opts.name
                if self.many:
                    namespace = self.opts.plural_name
                data = {namespace: data}
            return data


Our application schemas can now inherit from our custom schema class.

.. code-block:: python

    class UserSchema(NamespacedSchema):
        name = fields.String()
        email = fields.Email()

        class Meta:
            name = 'user'
            plural_name = 'users'

    ser = UserSchema()
    user = User('Keith', email='keith@stones.com')
    result = ser.dump(user)
    result.data  # {"user": {"name": "Keith", "email": "keith@stones.com"}}

