Configuration and deployment
============================

As your application grows more complex, you may find that you need to have different settings for
your development environment and your production environment. You may even have multiple
deployments that all need their own custom configuration.

For this purpose, Asphalt provides a command line interface that will read a YAML_ formatted
configuration file and run the application it describes.

.. _YAML: http://yaml.org/

Running the Asphalt launcher
----------------------------

Running the launcher is very straightfoward:

.. code-block:: bash

    asphalt run yourconfig.yaml [your-overrides.yml...]

Or alternatively:

    python -m asphalt run yourconfig.yaml [your-overrides.yml...]

What this will do is:

#. read all the given configuration files, starting from ``yourconfig.yaml``
#. merge the configuration files' contents into a single configuration dictionary using
    :func:`~asphalt.core.util.merge_config`
#. call :func:`~asphalt.core.runner.run_application` using the configuration dictionary as keyword
    arguments

Writing a configuration file
----------------------------

.. highlight:: yaml

A production-ready configuration file should contain at least the following options:

* ``component``: a dictionary containing the class name and keyword arguments for its constructor
* ``logging``: a dictionary to be passed to :func:`logging.config.dictConfig`

Suppose you had the following component class as your root component::

    class MyRootComponent(ContainerComponent):
        def __init__(self, components, data_directory: str):
            super().__init__(components)
            self.data_directory = data_directory

        async def start(ctx):
            self.add_component('mailer', backend='smtp')
            self.add_component('sqlalchemy')
            await super().start(ctx)

You could then write a configuration file like this::

    ---
    component:
      type: myproject:MyRootComponent
      data_directory: /some/file/somewhere
      components:
        mailer:
          host: smtp.mycompany.com
          ssl: true
        sqlalchemy:
          url: postgresql:///mydatabase
    max_threads: 20
    logging:
      version: 1
      disable_existing_loggers: false
      handlers:
        console:
          class: logging.StreamHandler
          formatter: generic
      formatters:
        generic:
            format: "%(asctime)s:%(levelname)s:%(name)s:%(message)s"
      root:
        handlers: [console]
        level: INFO

In the above configuration you have three top level configuration keys: ``component``,
``max_threads`` and ``logging``, all of which are directly passed to
:func:`~asphalt.core.runner.run_application` as keyword arguments.

The ``component`` section defines the type of the root component using the specially processed
``type`` option. You can either specify a setuptools entry point name (from the
``asphalt.components`` namespace) or a text reference like ``module:class`` (see
:func:`~asphalt.core.util.resolve_reference` for details). The rest of the keys in this section are
passed directly to the constructor of the ``MyRootComponent`` class.

The ``components`` section within ``component`` is processed in a similar fashion.
Each subsection here is a component type alias and its keys and values are the constructor
arguments to the relevant component class. The per-component configuration values are merged with
those provided in the ``start()`` method of ``MyRootComponent``. See the next section for a more
elaborate explanation.

With ``max_threads: 20``, the maximum number of threads in the event loop's default thread pool
executor is set to 20.

The ``logging`` configuration tree here sets up a root logger that prints all log entries of at
least ``INFO`` level to the console. You may want to set up more granular logging in your own
configuration file. See the
:ref:`Python standard library documentation <python:logging-config-dictschema>` for details.

Configuration overlays
----------------------

Component configuration can be specified on several levels:

* Hard-coded arguments to :meth:`~asphalt.core.component.ContainerComponent.add_component`
* First configuration file argument to ``asphalt run``
* Second configuration file argument to ``asphalt run``
* ...

Any options you specify on each level override or augment any options given on previous levels.
To minimize the effort required to build a working configuration file for your application, it is
suggested that you pass as many of the options directly in the component initialization code and
leave only deployment specific options like API keys, access credentials and such to the
configuration file.

With the configuration presented in the earlier paragraphs, the ``mailer`` component's constructor
gets passed three keyword arguments:

* ``backend='smtp'``
* ``host='smtp.mycompany.com'``
* ``ssl=True``

The first one is provided in the root component code while the other two options come from the YAML
file. You could also override the mailer backend in the configuration file if you wanted. The same
effect can be achieved programmatically by supplying the override configuration to the container
component via its ``components`` constructor argument. This is very useful when writing tests
against your application. For example, you might want to use the ``mock`` mailer in your test suite
configuration to test that the application correctly sends out emails (and to prevent them from
actually being sent to recipients!).

There is another neat trick that lets you easily modify a specific key in the configuration.
By using dotted notation in a configuration key, you can target a specific key arbitrarily deep in
the configuration structure. For example, to override the logging level for the root logger in the
configuration above, you could use an override configuration such as::

    ---
    logging.root.level: DEBUG

The keys don't need to be on the top level either, so the following has the same effect::

    ---
    logging:
        root.level: DEBUG

Performance tuning
------------------

Asphalt's core code and many third part components employ a number of potentially expensive
validation steps in its code. The performance hit of these checks is not a concern in development
and testing, but in a production environment you will probably want to maximize the performance.

To do this, you will want to disable Python's debugging mode by either setting the environment
variable ``PYTHONOPTIMIZE`` to ``1`` or (if applicable) running Python with the ``-O`` switch.
This has the effect of completely eliminating all ``assert`` statements and blocks starting with
``if __debug__:`` from the compiled bytecode.

When you want maximum performance, you'll also want to use the fastest available event loop
implementation. This can be done by specifying the ``event_loop_policy`` option in the
configuration file or by using the ``-l`` or ``--loop`` switch. The core library has built-in
support for the uvloop_ event loop implementation, which should provide a nice performance boost
over the standard library implementation.

.. _uvloop: http://magic.io/blog/uvloop-make-python-networking-great-again/
