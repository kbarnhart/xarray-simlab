.. _create_model:

Create and Modify Models
========================

Like the previous :doc:`framework` section, this section is useful
mostly for users who want to create new models from scratch or
customize existing models. Users who only want to run simulations from
existing models may skip this section.

As a simple example, we will start here from a model which numerically
solves the 1-d advection equation using the `Lax method`_. The equation
may be written as:

.. math::

   \frac{\partial u}{\partial t} + \nu \frac{\partial u}{\partial x} = 0

with :math:`u(x, t)` as the quantity of interest and where :math:`\nu`
is the velocity. The discretized form of this equation may be written
as:

.. math::

   u^{n+1}_i = \frac{1}{2} (u^n_{i+1} + u^n_{i-1}) - \nu \frac{\Delta t}{2 \Delta x} (u^n_{i+1} - u^n_{i-1})

Where :math:`\Delta x` is the fixed spacing between the nodes
:math:`i` of a uniform grid and :math:`\Delta t` is the step duration
between times :math:`n` and :math:`n+1`.

We could just implement this numerical model with a few lines of
Python / Numpy code, e.g., here below assuming periodic boundary
conditions and a Gaussian pulse as initial profile. We will show,
however, that it is very easy to refactor this code for using it with
xarray-simlab. We will also show that, while enabling useful features,
the refactoring still results in a short amount of readable code.

.. literalinclude:: scripts/advection_model_numpy.py

.. _`Lax method`: https://en.wikipedia.org/wiki/Lax%E2%80%93Friedrichs_method

Anatomy of a Process subclass
-----------------------------

Let's first wrap the code above into a single class named
``AdvectionLax1D`` decorated by :class:`~xsimlab.process`. Next we'll
explain in detail the content of this class.

.. literalinclude:: scripts/advection_model.py
   :lines: 3-32

Process interface
~~~~~~~~~~~~~~~~~

``AdvectionLax1D`` has some class attributes declared at the top,
which together form the process' "public" interface, i.e., all the
variables that we want to be publicly exposed by this process. Here we
use :func:`~xsimlab.variable` to add some metadata to each variable
of the interface.

We first may specify the labels of the dimensions expected for each
variable, which defaults to an empty tuple (i.e., a scalar value is
expected). In this example, variables ``spacing``, ``length``, ``loc``
and ``scale`` are all scalars, whereas ``x`` and ``u`` are both arrays
defined on the 1-dimensional :math:`x` grid. Multiple choices can also
be given as a list, like variable ``v`` which represents a velocity
field that can be either constant (scalar) or variable (array) in
space.

.. note::

   All variable objects also implicitly allow a time dimension.
   See section :doc:`run_model`.

Additionally, it is also possible to add a short ``description``
and/or custom metadata like units with the ``attrs`` argument.

Another important argument is ``intent``, which specifies how the
process deals with the value of the variable. By default,
``intent='in'`` means that the process just needs the value of the
variable for its computation ; this value should either be computed
elsewhere by another process or be provided by the user as model
input. By contrast, variables ``x`` and ``u`` have ``intent='out'``,
which means that the process ``AdvectionLax1D`` itself initializes and
computes a value for these two variables.

Process "runtime" methods
~~~~~~~~~~~~~~~~~~~~~~~~~

Beside its interface, the process ``AdvectionLax1D`` also implements
methods that will be called during simulation runtime:

- ``.initialize()`` will be called once at the beginning of a
  simulation. Here it is used to set the x-coordinate values of the
  grid and the initial values of ``u`` along the grid (Gaussian
  pulse).
- ``.run_step()`` will be called at each time step iteration and have
  the current time step duration as required argument. This is where
  the Lax method is implemented.
- ``.finalize_step()`` will be called at each time step iteration too
  but after having called ``run_step`` for all other processes (if
  any). Its intended use is mainly to ensure that state variables like
  ``u`` are updated consistently and after having taken snapshots.

A fourth method ``.finalize()`` could also be implemented, but it is
not needed in this case. This method is called once at the end of the
simulation, e.g., for some clean-up.

Getting / setting variable values
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For each variable declared as class attributes in ``AdvectionLax1D``
we can get their value (and/or set a value depending on their
``intent``) elsewhere in the class like if it was defined as regular
instance attributes, e.g., using ``self.u`` for variable ``u``.

.. note::

   In xarray-simlab it is safe to run multiple simulations
   concurrently: each simulation has its own process instances.

Beside variables declared in the process interface, nothing prevent us
from using regular attributes in process classes if needed. For
example, ``self.u1`` is set as a temporary internal state in
``AdvectionLax1D`` to wait for the "finalize step" stage before
updating :math:`u`.

Creating a Model instance
-------------------------

Creating a new :class:`~xsimlab.Model` instance is very easy. We just
need to provide a dictionary with the process class(es) that we want
to include in the model, e.g., with only the process created above:

.. literalinclude:: scripts/advection_model.py
   :lines: 35

That's it! Now we have different tools already available to inspect
the model (see section :doc:`inspect_model`). We can also use that
model with the xarray extension provided by xarray-simlab to create
new setups, run the model, take snapshots for one or more variables on
a given frequency, etc. (see section :doc:`run_model`).

Fine-grained process refactoring
--------------------------------

The model created above isn't very flexible. What if we want to change
the initial conditions? Use a grid with variable spacing? Add another
physical process impacting :math:`u` such as a source or sink term?
In all cases we would need to modify the class ``AdvectionLax1D``.

This framework works best if we instead split the problem into small
pieces, i.e., small process classes that we can easily combine and
replace in models.

The ``AdvectionLax1D`` process may for example be refactored into 4
separate processes:

- ``UniformGrid1D`` : grid creation
- ``ProfileU`` : update :math:`u` values along the grid at each time
  iteration
- ``AdvectionLax`` : perform advection at each time iteration
- ``InitUGauss`` : create initial :math:`u` values along the grid.

**UniformGrid1D**

This process declares all grid-related variables and computes
x-coordinate values.

.. literalinclude:: scripts/advection_model.py
   :lines: 38-47

Grid x-coordinate values only need to be set once at the beginning of
the simulation ; there is no need to implement ``.run_step()`` here.

**ProfileU**

.. literalinclude:: scripts/advection_model.py
   :lines: 50-62

``u_vars`` is declared as a :func:`~xsimlab.group` variable, i.e., an
iterable of all variables declared elsewhere that belong the same
group ('u_vars' in this case). In this example, it allows to further
add one or more processes that will also affect the evolution of
:math:`u` in addition to advection (see below).

Note also ``intent='inout'`` set for ``u``, which means that
``ProfileU`` updates the value of :math:`u` but still needs an initial
value from elsewhere.

**AdvectionLax**

.. literalinclude:: scripts/advection_model.py
   :lines: 65-83

``u_advected`` represents the effect of advection on the evolution of
:math:`u` and therefore belongs to the group 'u_vars'.

Computing values of ``u_advected`` requires values of variables
``spacing`` and ``u`` that are already declared in the
``UniformGrid1D`` and ``ProfileU`` classes, respectively.  Here we
declare them as :func:`~xsimlab.foreign` variables, which allows to
handle them like if these were the original variables. For example,
``self.grid_spacing`` in this class will return the same value than
``self.spacing`` in ``UniformGrid1D``.

**InitUGauss**

.. literalinclude:: scripts/advection_model.py
   :lines: 86-96

A foreign variable can also be used to set values for variables that
are declared in other processes, as for ``u`` here with
``intent='out'``.

**Refactored model**

We now have all the building blocks to create a more flexible model:

.. literalinclude:: scripts/advection_model.py
   :lines: 99-102

The order in which processes are given doesn't matter (it is a
dictionary). A computationally consistent order, as well as model
inputs among all declared variables, are both automatically figured
out when creating the Model instance.

In terms of computation and inputs, ``model2`` is equivalent to the
``model1`` instance created above ; it is just organized
differently.

Update existing models
----------------------

Between the two Model instances created so far, the advantage of
``model2`` over ``model1`` is that we can easily update the model --
change its behavior and/or add many new features -- without
sacrificing readability or losing the ability to get back to the
original, simple version.

**Example: adding a source term at a specific location**

For this we create a new process:

.. literalinclude:: scripts/advection_model.py
   :lines: 105-130

Some comments about this class:

- ``u_source`` belongs to the group 'u_vars' and therefore will be
  added to ``u_advected`` in ``ProfileU`` process.
- Methods and/or properties other than the reserved "runtime" methods
  may be added in a Process subclass, just like in any other Python
  class.
- Nearest node index and source rate array will be recomputed at each
  time iteration because variables ``loc`` and ``flux`` may both have
  a time dimension (variable source location and intensity), i.e.,
  ``self.loc`` and ``self.flux`` may both change at each
  time iteration.

In this example we also want to start with a flat, zero :math:`u`
profile instead of a Gaussian pulse. We create another (minimal)
process for that:

.. literalinclude:: scripts/advection_model.py
   :lines: 133-141

Using one command, we can then update the model with these new
features:

.. literalinclude:: scripts/advection_model.py
   :lines: 144-145

Compared to ``model2``, this new ``model3`` have a new process named
'source' and a replaced process 'init'.

**Removing one or more processes**

It is also possible to create new models by removing one or more
processes from existing Model instances, e.g.,

.. literalinclude:: scripts/advection_model.py
   :lines: 148

In this latter case, users will have to provide initial values of
:math:`u` along the grid directly as an input array.

.. note::

   Model instances are immutable, i.e., once created it is not
   possible to modify these instances by adding, updating or removing
   processes. Both methods ``.update_processes()`` and
   ``.drop_processes()`` always return new instances of ``Model``.
