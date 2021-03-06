MSBuildToolchain
================

.. warning::

    This is an **experimental** feature subject to breaking changes in future releases.

.. warning:

    Starting in Conan 1.32 ``write_toolchain_files()`` method and ``toolchain`` attribute have been
    deprecated. They will be removed in Conan 1.33, please use ``generate()`` instead of
    ``write_toolchain_files()`` and ``generate`` or ``generators = "MSBuildToolchain"`` instead of the
    ``toolchain`` attribute.

The ``MSBuildToolchain`` can be used in the ``generate()`` method:


.. code:: python

    from conans import ConanFile
    from conan.tools.microsoft import MSBuildToolchain

    class App(ConanFile):
        settings = "os", "arch", "compiler", "build_type"
        requires = "hello/0.1"
        generators = "msbuild"
        options = {"shared": [True, False]}
        default_options = {"shared": False}

        def generate(self):
            tc = MSBuildToolchain(self)
            tc.generate()


The ``MSBuildToolchain`` will generate two files after a ``conan install`` command or
before calling the ``build()`` method when the package is building in the cache:

- The main *conantoolchain.props* file, that can be used in the command line.
- A *conantoolchain_<config>.props* file, that will be conditionally included from the previous
  *conantoolchain.props* file based on the configuration, platform and toolset, e.g.:
  *conantoolchain_Release_x86_v140.props*

Every invocation to ``conan install`` with different configuration will create a new properties ``.props``
file, that will also be conditionally included. This allows to install different sets of dependencies,
then switch among them directly from the Visual Studio IDE.

The toolchain files can configure:

- The Visual Studio runtime (MT/MD/MTd/MDd), obtained from Conan input settings
- The C++ standard, obtained from Conan input settings


Generators
----------

The ``MSBuildToolchain`` only works with the ``msbuild`` generator.
Please, do not use other generators, as they can have overlapping definitions that can conflict.


Using the toolchain in developer flow
-------------------------------------

One of the advantages of using Conan toolchains is that they can help to achieve the exact same build
with local development flows, than when the package is created in the cache.

With the ``MSBuildToolchain`` it is possible to do:

.. code:: bash

    # Lets start in the folder containing the conanfile.py
    $ mkdir build && cd build
    # Install both debug and release deps and create the toolchain
    $ conan install ..
    $ conan install .. -s build_type=Debug
    # Add ``conantoolchain.props`` in your IDE to the project properties
    # No need to add the configuration .props files. This needs to be done only once
    # If you have dependencies, you will need to add the .props files of the dependencies
    # too, check the "msbuild" generator
    # Open Visual Studio IDE and build, switching configurations directly in the IDE


MSBuild build helper
---------------------

When using the toolchain feature, the ``MSBuild`` helper that is used in the ``build()`` method
will be a new, different one with new behavior.

.. warning::

    The new ``MSBuild`` helper that is used with toolchains is experimental and subject to
    breaking changes in the future

The ``MSBuild`` helper can be used like:

.. code:: python

    from conans import ConanFile, MSBuildToolchain, MSBuild

    class App(ConanFile):
        settings = "os", "arch", "compiler", "build_type"
        def generate(self):
            ...

        def build(self):
            msbuild = MSBuild(self)
            msbuild.build("MyProject.sln")

The ``MSBuild.build()`` method internally implements a call to ``msbuild`` like:


.. code:: bash

    $ <vcvars-cmd> && msbuild "MyProject.sln" /p:Configuration=<conf> /p:Platform=<platform>

Where:

- ``vcvars-cmd`` is calling the Visual Studio prompt that matches the current recipe ``settings``
- ``conf`` is the configuration, typically Release, Debug, which will be obtained from ``settings.build_type``
  but this will be configurable. Please open a `Github issue <https://github.com/conan-io/conan/issues>`_ if you want to define custom configurations.
- ``platform`` is the architecture, a mapping from the ``settings.arch`` to the common 'x86', 'x64', 'ARM', 'ARM64'.
  If your platform is unsupported, please report in `Github issues <https://github.com/conan-io/conan/issues>`_ as well:
