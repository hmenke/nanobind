.. _why:

.. cpp:namespace:: nanobind

Why another binding library?
============================

I started the `pybind11 <http://github.com/pybind/pybind11>`_ project back in
2015 to generate better C++/Python bindings for a project I had been working
on. Thanks to many amazing contributions by others, pybind11 has since become a
core dependency of software used across the world including flagship projects
like `PyTorch <https://pytorch.org>`_ and `Tensorflow
<https://www.tensorflow.org>`_. Every day, is is downloaded over 400'000 times.
Hundreds of contributed extensions and generalizations address use cases of
this diverse audience. However, all of this success also came with costs: the
complexity of the library grew tremendously, which had a negative impact on
efficiency.

Curiously, the situation now is reminiscent of 2015: binding generation with
existing tools (`Boost.Python <https://github.com/boostorg/python>`_, `pybind11
<http://github.com/pybind/pybind11>`_) is slow and produces enormous binaries
with overheads on runtime performance. At the same time, key improvements in
C++17 and Python 3.8 provide opportunities for drastic simplifications.
Therefore, I am starting *another* binding project. This time, the scope is
intentionally limited so that this doesn't turn into an endless cycle.

So what is different?
---------------------

nanobind is highly related to pybind11 and inherits most of its conventions
and syntax. The main difference is is a change in philosophy: pybind11 must
deal with *all of C++* to bind legacy codebases, while nanobind targets
a smaller C++ subset. *The codebase has to adapt to the binding tool and not
the other way around*, which allows nanobind to be simpler and faster. Pull
requests with extensions and generalizations to handle subtle fringe cases were
welcomed in pybind11, but they will likely be rejected in this project.

An overview of removed features is provided in a :ref:`separate section
<removed>`. Besides feature removal, the rewrite was also an opportunity to
address :ref:`long-standing performance issues <perf_improvements>` and add a
number of :ref:`major quality-of-live improvements <major_additions>` and
:ref:`smaller features <minor_additions>`.

.. _perf_improvements:

Performance improvements
------------------------

The :ref:`benchmark section <benchmarks>` evaluates the impact of the following
performance improvements:

- C++ objects are now co-located with the Python object whenever possible (less
  pointer chasing compared to pybind11). The per-instance overhead for wrapping
  a C++ type into a Python object shrinks by a factor of 2.3x. (pybind11: 56
  bytes, nanobind: 24 bytes.)

- C++ function binding information is now co-located with the Python
  function object (less pointer chasing).

- C++ type binding information is now co-located with the Python type object
  (less pointer chasing, fewer hashtable lookups).

- nanobind internally replaces ``std::unordered_map`` with a more efficient
  hash table (`tsl::robin_map <https://github.com/Tessil/robin-map>`_, which
  is included as a git submodule).

- function calls from/to Python are realized using `PEP 590 vector calls
  <https://www.python.org/dev/peps/pep-0590>`_, which gives a nice speed
  boost. The main function dispatch loop no longer allocates heap memory.

- pybind11 was designed as a header-only library, which is generally a good
  thing because it simplifies the compilation workflow. However, one major
  downside of this is that a large amount of redundant code has to be
  compiled in each binding file (e.g., the function dispatch loop and all of
  the related internal data structures). nanobind compiles a separate shared
  or static support library (``libnanobind``) and links it against the binding
  code to avoid redundant compilation. When using the CMake
  :cmake:command:`nanobind_add_module()` function, this all happens
  transparently.

- ``#include <pybind11/pybind11.h>`` pulls in a large portion of the STL
  (about 2.1 MiB of headers with Clang and libc++). nanobind minimizes STL
  usage to avoid this problem. Type casters even for for basic types like
  ``std::string`` require an explicit opt-in by including an extra header
  file (e.g. ``#include <nanobind/stl/string.h>``).

- pybind11 is dependent on *link time optimization* (LTO) to produce
  reasonably-sized bindings, which makes linking a build time bottleneck.
  With nanobind's split into a precompiled core library and minimal
  metatemplating, LTO is no longer such a big deal.

- nanobind maintains efficient internal data structures for lifetime management
  (needed for :cpp:class:`nb::keep_alive <keep_alive>`,
  :cpp:enumerator:`nb::rv_policy::reference_internal
  <rv_policy::reference_internal>`, the ``std::shared_ptr`` interface, etc.).
  With these changes, it is no longer necessary that bound types are
  weak-referenceable, which saves a pointer per instance.

.. _major_additions:

Major additions
---------------

nanobind includes a number of quality-of-live improvements for developers:

- **Tensors**: nanobind can exchange
  tensors with modern array
  programming frameworks. It uses
  either `DLPack
  <https://github.com/dmlc/dlpack>`_
  or the `buffer protocol
  <https://docs.python.org/3/c-api/buffer.html>`_
  to achieve *zero-copy* CPU/GPU
  tensor exchange with frameworks
  like `NumPy <https://numpy.org>`_,
  `PyTorch <https://pytorch.org>`_,
  `TensorFlow
  <https://www.tensorflow.org>`_,
  `JAX
  <https://jax.readthedocs.io>`_,
  etc. See the :ref:`section on
  tensors <tensors>` for details.

- **Stable ABI**: nanobind can target Python's `stable ABI interface
  <https://docs.python.org/3/c-api/stable.html>`_ starting with Python 3.12.
  This means that extension modules will be compatible with 
  future version of Python without having to compile separate binaries per interpreter. That vision is still relatively far out, however: it will require Python
  3.12+ to be widely deployed.

- **Leak warnings**: When the Python interpreter shuts down, nanobind reports
  instance, type, and function leaks related to bindings, which is useful for
  tracking down reference counting issues.  If these warnings are undesired,
  call :cpp:func:`nb::set_leak_warnings(false) <set_leak_warnings>`. nanobind
  also fully deletes its internal data structures when the Python interpreter
  terminates, which avoids memory leak reports in tools like *valgrind*.

- **Better docstrings**: pybind11 pre-renders docstrings while the binding code
  runs. In other words, every call to ``.def(...)`` to bind a function
  immediately creates the underlying docstring. When a function takes a C++
  type as parameter that is not yet registered in pybind11, the docstring will
  include a C++ type name (e.g. ``std::vector<int, std::allocator<int>>``),
  which can look rather ugly. pybind11 binding declarations must be arranged
  very carefully to work around this issue.

  nanobind avoids the issue altogether by not pre-rendering docstrings: they
  are created on the fly when queried. nanobind also has improved
  out-of-the-box compatibility with documentation generation tools like `Sphinx
  <https://www.sphinx-doc.org/en/master/>`_.

- **Low-level API**: nanobind exposes an optional low-level API to provide
  fine-grained control over diverse aspects including :ref:`instance creation
  <lowlevel>`, :ref:`type creation <typeslots>`, and it can store
  :ref:`supplemental data <supplement>` in types. The low-level API provides a
  useful escape hatch to pursue advanced use cases that were not foreseen in
  the design of this library.

.. _minor_additions:

Minor additions
---------------

The following lists minor-but-useful additions relative to pybind11.

- **Finding Python objects associated with a C++ instance**: In addition to all
  of the return value policies supported by pybind11, nanobind provides one
  additional policy named :cpp:enumerator:`nb::rv_policy::none
  <rv_policy::none>` that *only* succeeds when the return value is already a
  known/registered Python object. In other words, this policy will never
  attempt to move, copy, or reference a C++ instance by constructing a new
  Python object.

  The new :cpp:func:`nb::find() <find>` function encapsulates this behavior. It
  resembles :cpp:func:`nb::cast() <cast>` in the sense that it returns the
  Python object associated with a C++ instance. But while :cpp:func:`nb::cast()
  <cast>` will create that Python object if it doesn't yet exist,
  :cpp:func:`nb::find() <find>` will return a ``nullptr`` object. This function
  is useful to interface with Python's :ref:`cyclic garbage collector
  <cyclic_gc>`.

- **Parameterized wrappers**: The :cpp:class:`nb::handle_t\<T\> <handle_t>` type
  behaves just like the :cpp:class:`nb::handle <handle>` class and wraps a
  ``PyObject *`` pointer. However, when binding a function that takes such an
  argument, nanobind will only call the associated function overload when the
  underlying Python object wraps a C++ instance of type ``T``.

  Similarly, the :cpp:class:`nb::type_object_t\<T\> <type_object_t>` type
  behaves just like the :cpp:class:`nb::type_object <type_object>` class and
  wraps a ``PyTypeObject *`` pointer. However, when binding a function that
  takes such an argument, nanobind will only call the associated function
  overload when the underlying Python type object is a subtype of the C++ type
  ``T``.

- **Raw docstrings**: In cases where absolute control over docstrings is
  required (for example, so that complex cases can be parsed by a tool like
  `Sphinx <https://www.sphinx-doc.org>`_), the :cpp:class:`nb::raw_doc`
  attribute can be specified to functions. In this case, nanobind will *skip*
  generation of a combined docstring that enumerates overloads along with type
  information.

  Example:

  .. code-block:: cpp

     m.def("identity", [](float arg) { return arg; });
     m.def("identity", [](int arg) { return arg; },
           nb::raw_doc(
               "identity(arg)\n"
               "An identity function for integers and floats\n"
               "\n"
               "Args:\n"
               "    arg (float | int): Input value\n"
               "\n"
               "Returns:\n"
               "    float | int: Result of the identity operation"));

  Writing detailed docstrings in this way is rather tedious. In practice, they
  would usually be extracted from C++ headers using a tool like `pybind11_mkdoc
  <https://github.com/pybind/pybind11_mkdoc>`_ (which also works fine with
  nanobind despite the name).
