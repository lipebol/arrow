.. Licensed to the Apache Software Foundation (ASF) under one
.. or more contributor license agreements.  See the NOTICE file
.. distributed with this work for additional information
.. regarding copyright ownership.  The ASF licenses this file
.. to you under the Apache License, Version 2.0 (the
.. "License"); you may not use this file except in compliance
.. with the License.  You may obtain a copy of the License at

..   http://www.apache.org/licenses/LICENSE-2.0

.. Unless required by applicable law or agreed to in writing,
.. software distributed under the License is distributed on an
.. "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
.. KIND, either express or implied.  See the License for the
.. specific language governing permissions and limitations
.. under the License.

.. _benchmarks:

==========
Benchmarks
==========

Setup
=====

First install the :ref:`Archery <archery>` utility to run the benchmark suite.

Running the benchmark suite
===========================

The benchmark suites can be run with the ``benchmark run`` sub-command.

.. code-block:: shell

   # Run benchmarks in the current git workspace
   archery benchmark run
   # Storing the results in a file
   archery benchmark run --output=run.json

Sometimes, it is required to pass custom CMake flags, e.g.

.. code-block:: shell

   export CC=clang-8 CXX=clang++8
   archery benchmark run --cmake-extras="-DARROW_SIMD_LEVEL=NONE"

Additionally a full CMake build directory may be specified.

.. code-block:: shell

   archery benchmark run $HOME/arrow/cpp/release-build

Comparison
==========

One goal with benchmarking is to detect performance regressions. To this end,
``archery`` implements a benchmark comparison facility via the ``benchmark
diff`` sub-command.

In the default invocation, it will compare the current source (known as the
current workspace in git) with local main branch:

.. code-block:: shell

  archery --quiet benchmark diff --benchmark-filter=FloatParsing
  -----------------------------------------------------------------------------------
  Non-regressions: (1)
  -----------------------------------------------------------------------------------
                 benchmark            baseline           contender  change % counters
   FloatParsing<FloatType>  105.983M items/sec  105.983M items/sec       0.0       {}

  ------------------------------------------------------------------------------------
  Regressions: (1)
  ------------------------------------------------------------------------------------
                  benchmark            baseline           contender  change % counters
   FloatParsing<DoubleType>  209.941M items/sec  109.941M items/sec   -47.632       {}

For more information, invoke the ``archery benchmark diff --help`` command for
multiple examples of invocation.

Iterating efficiently
~~~~~~~~~~~~~~~~~~~~~

Iterating with benchmark development can be a tedious process due to long
build time and long run times. Multiple tricks can be used with
``archery benchmark diff`` to reduce this overhead.

First, the benchmark command supports comparing existing
build directories, This can be paired with the ``--preserve`` flag to
avoid rebuilding sources from zero.

.. code-block:: shell

   # First invocation clone and checkouts in a temporary directory. The
   # directory is preserved with --preserve
   archery benchmark diff --preserve

   # Modify C++ sources

   # Re-run benchmark in the previously created build directory.
   archery benchmark diff /tmp/arrow-bench*/{WORKSPACE,master}/build

Second, a benchmark run result can be saved in a json file. This also avoids
rebuilding the sources, but also executing the (sometimes) heavy benchmarks.
This technique can be used as a poor's man caching.

.. code-block:: shell

   # Run the benchmarks on a given commit and save the result
   archery benchmark run --output=run-head-1.json HEAD~1
   # Compare the previous captured result with HEAD
   archery benchmark diff HEAD run-head-1.json

Third, the benchmark command supports filtering suites (``--suite-filter``)
and benchmarks (``--benchmark-filter``), both options supports regular
expressions.

.. code-block:: shell

   # Taking over a previous run, but only filtering for benchmarks matching
   # `Kernel` and suite matching `compute-aggregate`.
   archery benchmark diff                                       \
     --suite-filter=compute-aggregate --benchmark-filter=Kernel \
     /tmp/arrow-bench*/{WORKSPACE,master}/build

Instead of rerunning benchmarks on comparison, a JSON file (generated by
``archery benchmark run``) may be specified for the contender and/or the
baseline.

.. code-block:: shell

   archery benchmark run --output=baseline.json $HOME/arrow/cpp/release-build
   git checkout some-feature
   archery benchmark run --output=contender.json $HOME/arrow/cpp/release-build
   archery benchmark diff contender.json baseline.json

Regression detection
====================

Writing a benchmark
~~~~~~~~~~~~~~~~~~~

1. The benchmark command will filter (by default) benchmarks with the regular
   expression ``^Regression``. This way, not all benchmarks are run by default.
   Thus, if you want your benchmark to be verified for regression
   automatically, the name must match.

2. The benchmark command will run with the ``--benchmark_repetitions=K``
   options for statistical significance. Thus, a benchmark should not override
   the repetitions in the (C++) benchmark's arguments definition.

3. Due to #2, a benchmark should run sufficiently fast. Often, when the input
   does not fit in memory (L2/L3), the benchmark will be memory bound instead
   of CPU bound. In this case, the input can be downsized.

4. By default, google's benchmark library will use the cputime metric, which
   is the sum of runtime dedicated on the CPU for all threads of the process.
   By contrast to realtime which is the wall clock time, e.g. the difference
   between end_time - start_time. In a single thread model, the cputime is
   preferable since it is less affected by context switching. In a multi thread
   scenario, the cputime will give incorrect result since it'll be inflated by
   the number of threads and can be far off realtime. Thus, if the benchmark is
   multi threaded, it might be better to use
   ``SetRealtime()``, see this `example <https://github.com/apache/arrow/blob/a9582ea6ab2db055656809a2c579165fe6a811ba/cpp/src/arrow/io/memory-benchmark.cc#L223-L227>`_.

Scripting
=========

``archery`` is written as a python library with a command line frontend. The
library can be imported to automate some tasks.

Some invocation of the command line interface can be quite verbose due to build
output. This can be controlled/avoided with the ``--quiet`` option or the
``--output=<file>`` can be used, e.g.

.. code-block:: shell

   archery benchmark diff --benchmark-filter=Kernel --output=compare.json ...
