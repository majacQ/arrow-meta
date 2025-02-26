# Errors in Arrow Analysis

There are broadly three kinds of errors that may arise from Arrow Analysis. This files gives an overview of the information tracked in each case, which shall form the basis for top-quality error messages.

## Parsing errors

These errors arise from `pre`, `post`, or `invariant` blocks which cannot be translated into SMT formulae. For example, we cannot translate method calls to SMT:

```kotlin
fun f(xs: List[String]): String {
  pre({ !xs.get(0).isEmpty() }) { ... }
  ...
}
```

The Kotlin compiler won't catch these errors in its own analysis phase (like it would do with a type error), since this is perfectly good Kotlin code. However, it seems desirable for the programmer to know that a particular language feature cannot be used in these blocks.

## Unsatisfiability errors

These errors embody the idea that "something should have been true, but it is not." There are three cases in which this may arise:

- `UnsatCallPre` (attached to the argument): the required pre-conditions for a (method, property, function) call are not satisfied. For example:

    ```kotlin
    val wrong = 1 / 0  // does not satisfy '0 != 0'
    ```

- `UnsatBodyPost` (attached to the return value): the post-condition declared in a function body is not true. For example:

    ```kotlin
    fun f(x: Int): Int {
      pre({ x >= 0 }) { "non-negative" }
      val r = x + x
      return r.post({ it > 1 }) { "greater than 1" }
      // does not satisfy 'x + x > 1'
    }
    ```

- `UnsatInvariants` (attached to the new value): the invariant declared for a mutable variable is not satisfied by the new value. For example:

    ```kotlin
    fun g(): Int {
      var r = 1 invariant { it > 0 }
      r = 0 // does not satisfy '0 > 0'
      ...
    }
    ```

### Information available

See `checkImplicationOf` for the code which produces the errors.

- The _one_ constraint name and Boolean formula which could not be satisfied.
- A _counter-example_ (also called a _model_), which is an assignment of values to variables which show a specific instance in which the constraint is false.
    - In the `f` function above in the `UnsatBodyPost` epigraph, one such counter-example is `x == 0`, since in that case `0 + 0 > 1` is false.
    - By looking at the values of the model for the arguments, we can derive one specific trace for which the function fails.

## Inconsistency errors

These errors embody the idea that "there's no possible way in which we may end up in this situation." Usually this means that the code is somehow unreachable. There are four cases in which this may arise:

- `InconsistentBodyPre`: the set of pre-conditions given to the function leaves no possible way to call the function. For example:

    ```kotlin
    fun h(x: Int): Int {
      pre({ x > 0 }) { "greater than 0" }
      pre({ x < 0 }) { "smaller than 0" }
      // no value can be both < 0 and > 0
      ...
    }
    ```

- `InconsistentConditions` (attached to a particular condition): the body of a branch is never executed, because the condition is hangs upon conflicts with the rest of the information about the function. For example, if a condition goes against a pre-condition:

    ```kotlin
    fun i(x: Int): Int {
      pre({ x > 0 }) { "greater than 0" }
      if (x == 0) {
        // 'x > 0' and 'x == 0' are incompatible
        // so this branch is unreachable
      } else {
        ...
      }
    }
    ```

- `InconsistentCallPost` (attached to the function call): the post-conditions gathered after calling a function imply that this function could not be called at all. _This is really uncommon in practice_.

- `InconsistentInvariants` (attached to a local declaration): there is no way in which the invariant attached to a declaration may be satisfied. For example:

    ```kotlin
    fun j(x: Int): Int {
      pre({ x > 0 }) { "greater than 0" }
      var v = 3 invariant { v > x && v < 0 }
    }

### Information available

See `addAndCheckConsistency` for the code which produces the errors.

- The last set of constraints which was added to the SMT solver.
    - This is not very useful when considering the next item.
- An _unsatisfiable core_, which is a subset of the formulas added to the solver since the beginning of the process, and which summarize the incompatibility.
    - During the whole analysis of `i` lots of constraints are added, but the real problem is between `x > 0` and `x == 0`.

## Additional information

During the analysis, a different _SMT variable name_ is assigned to each _subexpression_. For example, we may have:

```kotlin
f(g(2), h())
// a -> 2
// b -> g(2)
// c -> h()
// d -> f(g(2), h())
```

We can leverage this information to write better error messages. If we have a constraint which states `b > 0`, we can replace it with `g(2) > 0`.