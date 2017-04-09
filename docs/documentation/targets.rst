Labeled Data Container
=======================

Depending on the domain-specific problem and the data one is
working with, a data source may or may not contain what is known
as **targets**. A target (singular) is a piece of information
about a single observation that represents the desired output (or
correct answer) for that specific observation. If targets are
involved, then we find ourselves in the realm of supervised
learning. This includes both, classification (predicting
categorical output) and regression (predicting real-valued
output).

Dealing with targets in a generic way was quite a design
challenge for us. There are a lot of aspects to consider and
issues to address in order to achieve an extensible and flexible
package architecture, that can deal with labeled- and unlabeled
data sources equally well.

- Some data container may not contain any targets or even
  understand what targets are.

- There are labeled data sources that are not considered
  :ref:`container` (like many data iterators). A flexible package
  design needs a reasonably consistent API for both cases.

- The targets can be in a different data container than the
  features. For example it is quite common to store the features
  in a matrix :math:`X`, while the corresponding targets
  are stored in a separate vector :math:`\vec{y}`.

- For some data container, the targets are an intrinsic part of
  the observations. Furthermore, it might be the case that every
  data set has its own convention concerning which part of an
  observation represents the target. An example for such a data
  container is a ``DataFrame`` where one column denotes the
  target. The name/index of the target-column depends on the
  concrete data set, and is in general different for each
  ``DataFrame``. In other words, this means that for some data
  containers, the type itself does not know how to access a
  target. Instead it has to be a user decision.

- There are scenarios, where a data container just serves as an
  interface to some remote data set, or a big data set that is
  stored on the disk. If so, it is likely the case, that the
  targets are not part of the observations, but instead part of
  the data container metadata. An example would be a data
  container that represents a directory of images in the file
  system, in which each sub-directory contains the images of a
  single class. In that scenario, the targets are known from the
  directory names (i.e. the metadata). As such it would be far
  more efficient if the data container can make use of this
  information, instead of having to load an actual image from the
  disk just to access its target. Remember that targets are not
  only needed during training itself, but also for data
  partitioning and resampling.

The targets logic is in some ways a bit more complex than the
:func:`getobs` logic. The main reason for this is that while
:func:`getobs` was solely define for data containers, we want the
target logic to seamlessly support a wide variety of data sources
and data scenarios. In this document, however, we will only focus
on data sources that are considered :ref:`container`.

Query Target(s)
-------------------

The first question one may ask is, why would access pattern
framework need to "extract" the targets out of some data
container. After all, it would be simpler to just pass the
targets as an additional parameter. In fact, that is pretty much
how almost all other ML frameworks handle labeled data. The
reason why we diverge from tradition this is two-fold.

1. The set of access pattern that work on labeled data is really
   just a superset of the set of access pattern that work on
   unlabeled data. So by doing it our way, we avoid duplicate
   code.

2. The second - and more important - reason is that we decided
   that there is really no convincing argument for restricting
   the user input to either be in the form of one variable
   (unlabeled data), or two variables (for labeled data).
   In fact, we wanted to allow the same variable to contain the
   features as well as targets.

To that end we provide the function :func:`targets`. It can be
used to query the, well, all the targets of some given data
container or data subset.

.. function:: targets(data, [obsdim])

   Extract the concrete targets from `data` and return them.

   This function is eager in the sense that it will always call
   :func:`getobs` unless a custom method for :func:`gettargets`
   (see later) is implemented for the type of `data`. This will
   make sure that actual values are returned (in contrast to
   placeholders such as :class:`DataSubset` or ``SubArray``).

   In other words, the returned values must be in the form
   intended to be passed as-is to some resampling strategy or
   learning algorithm.

   :param data: The object representing a data container.

   :param obsdim: \
        Optional. If it makes sense for the type of `data`, then
        `obsdim` can be used to specify which dimension of `data`
        denotes the observations. It can be specified in a
        type-stable manner as a positional argument, or as a more
        convenient keyword parameter. See :ref:`obsdim` for more
        information.

In some cases we will see that invoking :func:`targets` just
seems to return the given data container unchanged. The reason
for this is simple. What :func:`targets` tries to do is return
the portion of the given data container that corresponds to the
targets. The function assumes that there *must* be targets of
some sorts (otherwise, why would you call a function called
"targets"?). If there is no decision to be made (e.g. there is
only a single vector to begin with), then the function simply
returns the result of :func:`getobs` for the given data.

.. code-block:: jlcon

   julia> targets([1,2,3,4])
   4-element Array{Int64,1}:
    1
    2
    3
    4

The above edge-case isn't really that informative for the main
functionality that :func:`targets` provides. The more interesting
behaviour can be seen for custom types and/or tuples. More
specifically, if ``data`` is a ``Tuple``, then the convention is
that the last element of the tuple contains the targets and the
function is recursed once (and only once).

.. code-block:: jlcon

   julia> targets(([1,2], [3,4]))
   2-element Array{Int64,1}:
    3
    4

   julia> targets(([1,2], ([3,4], [5,6])))
   ([3,4],[5,6])

What this shows us is that we can use tuples to create a labeled
data container out of two simple data containers. This is
particularly useful when working with arrays. Considering the
following situation, where we have a feature matrix ``X`` and a
corresponding target vector ``y``.

.. code-block:: jlcon

   julia> X = rand(2, 5)
   2×5 Array{Float64,2}:
    0.987618  0.365172  0.306373  0.540434  0.805117
    0.801862  0.469959  0.704691  0.405842  0.014829

   julia> y = [:a, :a, :b, :a, :b]
   5-element Array{Symbol,1}:
    :a
    :a
    :b
    :a
    :b

   julia> targets((X, y))
   5-element Array{Symbol,1}:
    :a
    :a
    :b
    :a
    :b

You may have noticed from the signature of :func:`targets`, that
there is no parameter for passing indices. This is no accident.
The purpose of :func:`targets` is not subsetting, it is to
extract the targets; no more, no less. If you wish to only query
the targets of a subset of some data container, you can use
:func:`targets` in combination with :func:`datasubset`.

.. code-block:: jlcon

   julia> targets(datasubset((X, y), 2:3))
   2-element Array{Symbol,1}:
    :a
    :b

If the type of the data itself is not sufficient information to
be able to extract the targets, one can specify a
target-extraction-function ``fun`` that is to be applied to each
observation. This function must be passed as the first parameter
to :func:`targets`.

.. function:: targets(fun, data, [obsdim]) -> Vector

   Extract the concrete targets from the observations in `data`
   by applying `fun` on each observation individually. The
   extracted targets are returned as a ``Vector``, which
   preserves the order of the observations from `data`.

   :param fun: \
        A callable object (usually a function) that should
        be applied to each observation individually in order to
        extract or compute the target for that observation.

   :param data: The object representing a data container.

   :param obsdim: \
        Optional. If it makes sense for the type of `data`, then
        `obsdim` can be used to specify which dimension of `data`
        denotes the observations. It can be specified in a
        typestable manner as a positional argument, or as a more
        convenient keyword parameter. See :ref:`obsdim` for more
        information.

A great example for a data source, that stores the features and
the targets in the same manner, is a ``DataFrame``. There is no
clear convention what column of the table denotes the targets; it
depends on the data set. As such, we require a data-specific
target-extraction-function. Consider the following example using
a toy ``DataFrame`` (see :ref:`dataframe` to make the following
code work). For this particular data frame we know that the
column ``:y`` contains the targets.

.. code-block:: jlcon

   julia> df = DataFrame(x1 = rand(5), x2 = rand(5), y = [:a,:a,:b,:a,:b])
   5×3 DataFrames.DataFrame
   │ Row │ x1        │ x2       │ y │
   ├─────┼───────────┼──────────┼───┤
   │ 1   │ 0.176654  │ 0.821837 │ a │
   │ 2   │ 0.0397664 │ 0.894399 │ a │
   │ 3   │ 0.390938  │ 0.29062  │ b │
   │ 4   │ 0.582912  │ 0.509047 │ a │
   │ 5   │ 0.407289  │ 0.113006 │ b │

   julia> targets(row->row[1,:y], df)
   5-element Array{Symbol,1}:
    :a
    :a
    :b
    :a
    :b

Another use-case for specifying an extraction function, is to
discretize some continuous regression targets. We will see later,
when we start discussing higher-level functions, how this can be
useful in order to over- or under-sample the data set (see
:func:`oversample` or :func:`undersample`).

.. code-block:: jlcon

   julia> targets(x -> (x > 0.7), rand(6))
   6-element Array{Bool,1}:
     true
    false
     true
    false
     true
     true

Note that if this optional first parameter (i.e. the extraction
function) is passed to :func:`targets`, it will always be applied
to the observations, and **not** the container. In other words,
the first parameter is applied to each observation individually
and not to the data as a whole. In general this means that the
return type changes drastically even if passing a no-op function.

.. code-block:: jlcon

   julia> X = rand(2, 3)
   2×3 Array{Float64,2}:
    0.105307   0.58033   0.724643
    0.0433558  0.116124  0.89431

   julia> y = [1 3 5; 2 4 6]
   2×3 Array{Int64,2}:
    1  3  5
    2  4  6

   julia> targets((X,y))
   2×3 Array{Int64,2}:
    1  3  5
    2  4  6

   julia> targets(x->x, (X,y))
   3-element Array{Array{Int64,1},1}:
    [1,2]
    [3,4]
    [5,6]

The optional parameter ``obsdim`` can be used to specify which
dimension denotes the observations, if that concept makes sense
for the type of ``data``.

.. code-block:: jlcon

   julia> X = [1 0; 0 1; 1 0]
   3×2 Array{Int64,2}:
    1  0
    0  1
    1  0

   julia> targets(indmax, X, obsdim=1)
   3-element Array{Int64,1}:
    1
    2
    1

   julia> targets(indmax, X, ObsDim.First())
   3-element Array{Int64,1}:
    1
    2
    1

Note how ``obsdim`` can either be provided using type-stable
positional arguments from the namespace ``ObsDim``, or by using a
more flexible and convenient keyword argument. See :ref:`obsdim`
for more information on that topic.

Iterate over Targets
---------------------

In some situations, one only wants to iterate over the targets,
instead of querying all of them at once. In those scenarios it
would be beneficial to avoid allocation temporary memory all
together. To that end we provide the function :func:`eachtarget`,
which returns a ``Base.Generator``, that when iterated over
returns each target in ``data`` once and in the correct order.

.. function:: eachtarget([fun], data, [obsdim]) -> Generator

   Return a ``Base.Generator`` that iterates over all targets in
   `data` once and in the right order. If `fun` is provided it
   will be applied to each observation in data.

   :param fun: \
        Optional. A callable object (usually a function) that
        should be applied to each observation individually in
        order to extract or compute the target for that
        observation.

   :param data: The object representing a data container.

   :param obsdim: \
        Optional. If it makes sense for the type of `data`, then
        `obsdim` can be used to specify which dimension of `data`
        denotes the observations. It can be specified in a
        typestable manner as a positional argument, or as a more
        convenient keyword parameter. See :ref:`obsdim` for more
        information.

The function :func:`eachtarget` behaves very similar to
:func:`targets`. For example, if you pass it a ``Tuple`` of data
container, then it will assume that the last tuple element
contains the targets.

.. code-block:: jlcon

   julia> iter = eachtarget(([1,2], [3,4]))
   Base.Generator{UnitRange{Int64},MLDataPattern.##79#80{2,Tuple{Array{Int64,1},Array{Int64,1}},Tuple{LearnBase.ObsDim.Last,LearnBase.ObsDim.Last}}}(MLDataPattern.#79,1:2)

   julia> collect(iter)
   2-element Array{Int64,1}:
    3
    4

The one big difference to :func:`targets` is that
:func:`eachtarget` will always iterate over the targets one
observation at a time, regardless whether or not an extraction
function is provided.

.. code-block:: jlcon

   julia> iter = eachtarget([1 3 5; 2 4 6])
   Base.Generator{UnitRange{Int64},MLDataPattern.##72#73{Array{Int64,2},LearnBase.ObsDim.Last}}(MLDataPattern.#72,1:3)

   julia> collect(iter)
   3-element Array{Array{Int64,1},1}:
    [1,2]
    [3,4]
    [5,6]

   julia> targets([1 3 5; 2 4 6]) # as comparison
   2×3 Array{Int64,2}:
    1  3  5
    2  4  6

Of course, it is also possible to work with any other type of
data source that is considered a :ref:`container`. Consider the
following example using a toy ``DataFrame`` (see :ref:`dataframe`
to make the following code work). For this particular data frame
we will assume that the column ``:y`` contains the targets.

.. code-block:: jlcon

   julia> df = DataFrame(x1 = rand(5), x2 = rand(5), y = [:a,:a,:b,:a,:b])
   5×3 DataFrames.DataFrame
   │ Row │ x1        │ x2       │ y │
   ├─────┼───────────┼──────────┼───┤
   │ 1   │ 0.176654  │ 0.821837 │ a │
   │ 2   │ 0.0397664 │ 0.894399 │ a │
   │ 3   │ 0.390938  │ 0.29062  │ b │
   │ 4   │ 0.582912  │ 0.509047 │ a │
   │ 5   │ 0.407289  │ 0.113006 │ b │

   julia> iter = eachtarget(row->row[1,:y], df)
   Base.Generator{MLDataPattern.ObsView{MLDataPattern.DataSubset{DataFrames.DataFrame,Int64,LearnBase.ObsDim.Undefined},...

   julia> collect(iter)
   5-element Array{Symbol,1}:
    :a
    :a
    :b
    :a
    :b

Just like for :func:`targets`, the optional parameter ``obsdim``
can be used to specify which dimension denotes the observations,
if that concept makes sense for the type of the given data.

.. code-block:: jlcon

   julia> X = [1 0; 0 1; 1 0]
   3×2 Array{Int64,2}:
    1  0
    0  1
    1  0

   julia> iter = eachtarget(indmax, X, obsdim = 1)
   Base.Generator{MLDataPattern.ObsView{SubArray{Int64,1,Array{Int64,2},Tuple{Int64,Colon},true},Array{Int64,2},LearnBase.ObsDim.Constant{1}},...

   julia> collect(iter)
   3-element Array{Int64,1}:
    1
    2
    1

.. _customtargets:

Support for Custom Types
--------------------------

Any labeled data container has the option to customize the
behaviour of :func:`targets`. The emphasis here is on "option",
because it is not required by the interface itself. Aside from
leaving the default behaviour, there are two ways to customize
the logic behind :func:`targets`.

1. Implement ``LearnBase.gettargets`` for the **data container**
   type. This will bypasses the function :func:`getobs` entirely,
   which can significantly improve the performance.

2. Implement ``LearnBase.gettarget`` for the **observation**
   type, which is applied on the result of :func:`getobs`. This
   is useful when the observation itself contains the target.

Let us consider two example scenarios that benefit from
implementing custom methods. The first one for
``LearnBase.gettargets``, and the second one for
``LearnBase.gettarget``. Note again that these functions are
internal and only intended to be *extended* by the user (and
**not** called). A user should not use them directly but instead
always call :func:`targets`.

Example 1: Custom Directory Based Image Source
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Let's say you want to write a custom data container that
describes a directory on your hard-drive. Each sub-directory is
expected to contain a set of large images that belong to a single
class (the directory name). This kind of data container only
loads the images itself if they are actually needed (so on
:func:`getobs`). The targets, however, would technically be
available in the memory at all times, since it is part of the
metadata.

To "simulate" such a scenario, let us define a dummy type that
represents the idea of such a data container for which each
observation is expensive to access, but where the corresponding
targets are available in some member variable.

.. code-block:: julia

   using LearnBase

   immutable DummyDirImageSource
       targets::Vector{String}
   end

   LearnBase.getobs(::DummyDirImageSource, i) = error("expensive computation triggered")

   LearnBase.nobs(data::DummyDirImageSource) = length(data.targets)

Naturally, we would like to avoid calling :func:`getobs` if at
all possible. While we can't avoid calling :func:`getobs` when we
actually need the data, we could avoid it when we only require
the targets (for example for data partitioning or resampling).
This is because in this example, the targets are part of the
metadata that is always loaded. We can make use of this fact by
implementing a custom method for ``LearnBase.gettargets``.

.. code-block:: julia

   LearnBase.gettargets(data::DummyDirImageSource, i) = data.targets[i]

By defining this method, the function :func:`targets` can now
query the targets efficiently by looking them up in the member
variable. In other words it allows to provide the targets of some
observation(s) without ever calling :func:`getobs`. This even
works seamlessly in combination with :func:`datasubset`.

.. code-block:: jlcon

   julia> source = DummyDirImageSource(["malign", "benign", "benign", "malign", "benign"])
   DummyDirImageSource(String["malign","benign","benign","malign","benign"])

   julia> targets(source)
   5-element Array{String,1}:
    "malign"
    "benign"
    "benign"
    "malign"
    "benign"

   julia> targets(datasubset(source, 3:4))
   2-element Array{String,1}:
    "benign"
    "malign"

Note however, that calling :func:`targets` with a
target-extraction-function will still trigger :func:`getobs`.
This is expected behaviour, since the extraction function is
intended to "extract" the target from each actual observation.

.. code-block:: jlcon

   julia> targets(x->x, source)
   ERROR: expensive computation triggered

Example 2: Symbol Support for DataFrames.jl
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``DataFrame`` are a kind of data container, where the targets are
as much part of the data as the features are (in contrast to
Example 1). Furthermore, each observation is itself also a
``DataFrame``. Before we start, let us implement the required
:ref:`container` interface.

.. code-block:: julia

   using DataFrames, LearnBase

   LearnBase.getobs(df::DataFrame, idx) = df[idx,:]

   LearnBase.nobs(df::DataFrame) = nrow(df)

Here we are fine with :func:`getobs` being called, since we need
to access the actual ``DataFrame`` anyway. However, we still need
to specify which column actually describes the features. This can
be done generically by specifying a target-extraction-function.

.. code-block:: jlcon

   julia> df = DataFrame(x1 = rand(5), x2 = rand(5), y = [:a,:a,:b,:a,:b])
   5×3 DataFrames.DataFrame
   │ Row │ x1       │ x2        │ y │
   ├─────┼──────────┼───────────┼───┤
   │ 1   │ 0.226582 │ 0.0997825 │ a │
   │ 2   │ 0.504629 │ 0.0443222 │ a │
   │ 3   │ 0.933372 │ 0.722906  │ b │
   │ 4   │ 0.522172 │ 0.812814  │ a │
   │ 5   │ 0.505208 │ 0.245457  │ b │

   julia> targets(row->row[1,:y], df)
   5-element Array{Symbol,1}:
    :a
    :a
    :b
    :a
    :b

Alternatively, we could also implement a convenience syntax by
overloading ``LearnBase.gettarget``.

.. code-block:: julia

   LearnBase.gettarget(col::Symbol, df::DataFrame) = df[1, col]

This now allows us to call ``targets(:Y, dataframe)``. While not
strictly necessary in this case, it can be quite useful for
special types of observations, such as ``ImageMeta``.

.. code-block:: jlcon

   julia> targets(:y, df)
   5-element Array{Symbol,1}:
    :a
    :a
    :b
    :a
    :b

We could even implement the default assumption, that the last
column denotes the targets unless otherwise specified.

.. code-block:: julia

   LearnBase.gettarget(df::DataFrame) = df[1, end]

Note that this might not be a good idea for a ``DataFrame`` in
particular. The purpose of this exercise is solely to show what
is possible.

.. code-block:: jlcon

   julia> targets(df)
   5-element Array{Symbol,1}:
    :a
    :a
    :b
    :a
    :b