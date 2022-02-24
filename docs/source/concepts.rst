.. _concepts:

Concepts
========

Machine learning pipelines, or even complex data pipelines, are made up of several *components.* For instance:

.. image:: images/toy-ml-pipeline-diagram.svg

Keeping track of data flow in and out of these components can be tedious, especially if multiple people are collaborating on the same end-to-end pipeline.This is because in ML pipelines, *different* artifacts are produced (inputs and outputs) when the *same* component is run more than once.

Knowing data flow is a precursor to debugging issues in data pipelines. ``mltrace`` also determines whether components of pipelines are stale.

Data model
^^^^^^^^^^

The two prominent client-facing abstractions are the :py:class:`~mltrace.Component` and :py:class:`~mltrace.ComponentRun` abstractions.

:py:class:`~mltrace.Test`
"""""""""

The ``Test`` abstraction represents some reusable computation to perform on component inputs and outputs. Defining a ``Test`` is similar to writing a unit test:

.. code-block :: python

    from mltrace import Test

    class OutliersTest(Test):
        def __init__(self):
            super().__init__(name='outliers')

        def testSomething(self; df: pd.DataFrame):
            ....
        
        def testSomethingElse(self; df: pd.DataFrame):
            ....


Tests can be defined and passed to components as arguments, as described in the section below.

:py:class:`mltrace.Component`
"""""""""

The ``Component`` abstraction represents a stage in a pipeline and its static metadata, such as:

* name
* description
* owner
* tags (optional list of string values to reference the component by)
* tests

Tags are generally useful when you have multiple components in a higher-level stage. For example, ETL computation could consist of different components such as "cleaning" or "feature generation." You could create the "cleaning" and "feature generation" components with the tag ``etl`` and then easily query component runs with the ``etl`` tag in the UI.

Components have a life-cycle:

* ``c = Component(...)``: construction of the component object
* ``c.beforeTests``: a list of ``Tests`` to run before the component is run
* ``c.run``: a decorator for a user-defined function that represents the component's computation
* ``c.afterTests``: a list of ``Tests`` to run after the component is run 

Putting it all together, we can define our own component:

.. code-block :: python

    from mltrace import Component

    class Featuregen(Component):
        def __init__(self, beforeTests=[], afterTests=[OutliersTest]):

        super().__init__(
            name="featuregen",
            owner="spark-gymnast",
            description="Generates features for high tip prediction problem",
            tags=["nyc-taxicab"],
            beforeTests=beforeTests,
            afterTests=afterTests,
        )
    

And in our main application code, we can decorate any feature generation function:

.. code-block :: python

    @Featuregen().run
    def generateFeatures(df: pd.DataFrame):
        # Generate features
        df = ...
        return df

See the next page for a more in-depth tutorial on instrumenting a pipeline.

:py:class:`mltrace.ComponentRun`
"""""""""

The ``ComponentRun`` abstraction represents an instance of a ``Component`` being run. Think of a ``ComponentRun`` instance as an object storing *dynamic* metadata for a ``Component``, such as:

* start timestamp
* end timestamp
* inputs
* outputs
* git hash
* source code
* dependencies (you do not need to manually declare)

If you dig into the codebase, you will find another abstraction, the :py:class:`~mltrace.IOPointer`. Inputs and outputs to a ``ComponentRun`` are stored as ``IOPointer`` objects. You do not need to explicitly create an ``IOPointer`` -- the abstraction exists so that ``mltrace`` can easily find and store dependencies between ``ComponentRun`` objects.

You will not need to explicitly define all of these variables, nor do you have to create instances of a ``ComponentRun`` yourself. See the next section for logging functions and an example.

.. _Staleness Overview:

Staleness
^^^^^^^^^^

We define a component run as "stale" if it may need to be rerun. Currently, ``mltrace`` detects two types of staleness in component runs:

1. A significant number of days (default 30) have passed between when a component run's inputs were generated and the component is run
2. At the time a component is run, its dependencies have fresher runs that began before the component run started

We are working on "data drift" as another measure of staleness.

.. _Reviewing Overview:

Reviewing erroneous outputs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Oftentimes there is a bug or error in some output of a pipeline that surfaces after the output has been produced. ML and data bugs are extra elusive because it can take a nontrivial number of mispredicted or buggy outputs to indicate that there is actually an issue with the pipeline. Given a set of erroneous outputs, it can be challenging to know where to begin debugging! Fortunately, ``mltrace`` can help with this.

The idea here is to identify the common ``ComponentRun`` s used in producing the erroneous outputs, as these might provide a good suggestion for what component to debug first or artifacts (inputs and outputs) to dive into. See steps on how to use the reviewer tool in the :ref:`querying` section.


