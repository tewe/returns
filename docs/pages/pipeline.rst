Pipelines
=========

The main idea behind functional programming is functional composition.

We provide several tools to make
composition easy, readable, pythonic, and useful.

Let's start with the first one.


.. _pipe:

pipe
----

``pipe`` is an easy way to compose functions together.
Let's see an example.

.. code:: python

  from returns.pipeline import pipe

  pipe(1, str, lambda x: x + 'b', str.upper)
  # => Will be equal to: `1B`

There's also a way to compose containers together:

.. code:: python

  from returns.pipeline import pipe
  from returns.result import Result
  from returns.functions import box

  def regular_function(arg: int) -> float:
      ...

  def returns_container(arg: float) -> Result[complex, ValueError]:
      ...

  def also_returns_container(args: complex) -> Result[str, ValueError]:
      ...

  pipe(
      1,  # we provide the initial value itself as a first argument
      regular_function,  # composes easily
      returns_container,  # also composes easily, but returns a container...
      # So we need to `box` the next function to allow it to consume
      # the container from the previous step.
      box(also_returns_container),
  )
  # => Will return `Result[str, ValueError]` as declared in the last step

See also :func:`returns.io.IO.lift` which is also extremely
helpful for ``IO`` composition.

Limitations
~~~~~~~~~~~

But, composition with ``pipe`` is limited to two things:

1. It only allows to pipe up to 7 functions.
   If you need more - send a PR with the type annotations.
   Python cannot figure things out by itself.
2. It is flexible. Sometimes you might need more power.
   Use ``@pipeline`` in this case!


.. _pipeline:

pipeline
--------

What is a ``pipeline``?
It is a more user-friendly syntax to work with containers
that support both async and regular functions.

Consider this task.
We were asked to create a method
that will connect together a simple pipeline of three steps:

1. We validate passed ``username`` and ``email``
2. We create a new ``Account`` with this data, if it does not exists
3. We create a new ``User`` associated with the ``Account``

And we know that this pipeline can fail in several places:

1. Wrong ``username`` or ``email`` might be passed, so the validation will fail
2. ``Account`` with this ``username`` or ``email`` might already exist
3. ``User`` creation might fail as well,
   since it also makes an ``HTTP`` request to another micro-service deep inside

Here's the code to illustrate the task.

.. code:: python

  from returns.result import Result, Success, Failure, pipeline

  class CreateAccountAndUser(object):
      """Creates new Account-User pair."""

      # TODO: we need to create a pipeline of these methods somehow...

      # Protected methods

      def _validate_user(
          self, username: str, email: str,
      ) -> Result['UserSchema', str]:
          """Returns an UserSchema for valid input, otherwise a Failure."""

      def _create_account(
          self, user_schema: 'UserSchema',
      ) -> Result['Account', str]:
          """Creates an Account for valid UserSchema's. Or returns a Failure."""

      def _create_user(
          self, account: 'Account',
      ) -> Result['User', str]:
          """Create an User instance. If user already exists returns Failure."""

Using bind technique
~~~~~~~~~~~~~~~~~~~~

We can implement this feature using a traditional ``bind`` method.

.. code:: python

  class CreateAccountAndUser(object):
      """Creates new Account-User pair."""

      def __call__(self, username: str, email: str) -> Result['User', str]:
          """Can return a Success(user) or Failure(str_reason)."""
          return self._validate_user(username, email).bind(
              self._create_account,
          ).bind(
              self._create_user,
          )

      # Protected methods
      # ...

And this will work without any problems.
But, is it easy to read a code like this? **No**, it is not.

What alternative we can provide?
:ref:`pipe` and :ref:`@pipeline <pipeline>`!
Read more about them if you want to compose your containers easily.

Using @pipeline
~~~~~~~~~~~~~~~

``@pipeline`` is a very powerful tool to compose things.
Let's see an example.

.. code:: python

  class CreateAccountAndUser(object):
      """Creates new Account-User pair."""

      @pipeline
      def __call__(self, username: str, email: str) -> Result['User', str]:
          """Can return a Success(user) or Failure(str_reason)."""
          user_schema = self._validate_user(username, email).unwrap()
          account = self._create_account(user_schema).unwrap()
          return self._create_user(account)

      # Protected methods
      # ...

Let's see how this new ``.unwrap()`` method works:

- if you result is ``Success`` it will return its inner value
- if your result is ``Failure`` it will raise a ``UnwrapFailedError``

And that's where ``@pipeline`` decorator becomes in handy.
It will catch any ``UnwrapFailedError`` during the pipeline
and then return a simple ``Failure`` result.

.. mermaid::
   :caption: Pipeline execution.

   sequenceDiagram
      participant pipeline
      participant validation
      participant account creation
      participant user creation

      pipeline->>validation: runs the first step
      validation-->>pipeline: returns Failure(validation message) if fails
      validation->>account creation: passes Success(UserSchema) if valid
      account creation-->>pipeline: return Failure(account exists) if fails
      account creation->>user creation: passes Success(Account) if valid
      user creation-->>pipeline: returns Failure(http status) if fails
      user creation-->>pipeline: returns Success(user) if user is created

See, do notation allows you to write simple yet powerful pipelines
with multiple and complex steps.
And at the same time the produced code is simple and readable.

Limitations
~~~~~~~~~~~

There's currently a typing-related issue with ``Result``:
you can unwrap wrong failure instance.
And the returning value will be different.

.. code:: python

  from returns.result import Result
  from returns.pipeline import pipeline

  @pipeline
  def example() -> Result[int, str]:
      other: Result[int, Exception]
      new_value = other.unwrap() + 1  # hidden boom!
      return Success(new_value)

Since ``mypy`` cannot know the context of ``.unwrap()`` method - it cannot
really tell is it allowed to unwrap a value or not.

In this case ``other`` might fail
and ``Result[int, Exception]`` might be returned.

What to do to minimize the effect?

1. Always stick to the same error type in your ``@pipeline`` results
2. Unit test things
3. Write a custom ``mypy`` plugin to check that and submit a PR :)


is_successful
-------------

:func:`is_succesful <returns.functions.is_successful>` is used to
tell whether or not your result is a success.
We treat only treat types that does not throw as a successful ones,
basically: :class:`Success <returns.result.Success>`.

.. code:: python

  from returns.result import Success, Failure, is_successful

  is_successful(Success(1))
  # => True

  is_successful(Failure('text'))
  # => False


Further reading
---------------

- https://dry-rb.org/gems/dry-monads/do-notation/
- https://github.com/gcanti/fp-ts/blob/master/src/pipeable.ts
- https://en.wikibooks.org/wiki/Haskell/do_notation
- https://wiki.haskell.org/Do_notation_considered_harmful


API Reference
-------------

.. autofunction:: returns.pipeline.pipe

.. automodule:: returns.pipeline
   :members: