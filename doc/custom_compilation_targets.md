Creating custom compilation targets
===================================

In RulerZ, we call a *compilation target* the class used to generate code to a
given target. For instance, we say that the class [`RulerZ\Target\Solarium\Solarium`](https://github.com/K-Phoen/rulerz/blob/master/src/Target/Solarium/Solarium.php)
targets Solr as it is responsible for compiling rules to Solr queries.

The generated program is called the executor. It is a specialized piece of code
that know how to execute a single rule, for a specific target. Each
rule/compilation pair will produce its own executor.

## Implementing the compilation target

Compilation targets are described by the [`RulerZ\Compiler\CompilationTarget`](https://github.com/K-Phoen/rulerz/blob/master/src/Compiler/CompilationTarget.php)
interface.

This interface has several methods but two of them are particularly important:
* `supports($target, $mode)`: this method is used by RulerZ to determine which
   compilation target will be used for the given `$target`. The first to return
   `true` will be used ;
* `compile(Rule $rule, Context $compilationContext)`: this method is used to
   compile a rule to a specialized executor. The rule is already *parsed* by
   RulerZ and is given as an AST.

Most of the time, implementing the `supports()` method is pretty straightforward
and only a few `instanceof` checks are needed. For instance, the Solarium
compilation target be used for targets of type `\Solarium\Client`.

```php
class Solarium
{
    // …

    public function supports($target, $mode)
    {
        return $target instanceof \Solarium\Client;
    }
}
```

The *hard* work begins with the implementation of the compilation itself. The
`compile()` method receives an AST representing the rule, as an instance of
`RulerZ\Model\Rule`. The whole point of the compilation target is to transform
this AST in an instance of `\RulerZ\Model\Executor`, which represents the
specialized executor and holds its code and a few other useful information.

To ease this compilation process, RulerZ provides an [`AbstractCompilationTarget`](https://github.com/K-Phoen/rulerz/blob/master/src/Target/AbstractCompilationTarget.php)
class that can be used as a base for compilation targets.
If you choose to use the `AbstractCompilationTarget` class, you will have to
implement a `function createVisitor(Context $context)` method.

Rule visitors implement the [`\RulerZ\Compiler\RuleVisitor`](https://github.com/K-Phoen/rulerz/blob/master/src/Compiler/RuleVisitor.php)
interface and follow the *visitor* pattern. Their goal is to browse the AST and
generate equivalent code for the targeted platform.
Once again, RulerZ provides a [`GenericVisitor`](https://github.com/K-Phoen/rulerz/blob/master/src/Target/GenericVisitor.php)
that implements a few common operations.

## Implementing the executor

Executors are described by the [`RulerZ\Executor\Executor`](https://github.com/K-Phoen/rulerz/blob/master/src/Executor/Executor.php)
interface.

They have to implement three methods:
* `applyFilter()`: which applies a filter on the given target ;
* `filter()`: which applies a filter on the target, and returns the
  corresponding results ;
* `satisfies()`: which indicates if the given target satisfies the rule.

Their skeleton is generated by the compiled and the placeholders are filled with
data contained in `\RulerZ\Model\Executor` instances:

```php
# ignore
namespace RulerZ\Compiled\Executor;

use RulerZ\Executor\Executor;

class {{ name }} implements Executor
{
    {{ traits }

    {{ additional code }}

    // {{ original rule }}
    protected function execute(\$target, array \$operators, array \$parameters)
    {
        return {{ compiled rule }};
    }
}
```

The ``{{ compiled rule }}`` placeholder will be filled by the result of the
visitor used in the compilation process (the compiled rule).

Most of the time (ie: always), all the compiled generated executors for a
given target will be identical, except for the "compiled rule" part.
That's why RulerZ uses traits to put the generic code on one side and the
specialized code on another. These traits are listed in the compilation target
implementation, by the `getExecutorTraits()` method.

In these traits, we typically find implementations of the methods required by the
`Executor` interface.
For instance, this method is from the SolariumFilterTrait:

```php
class Executor
{
    // …

    public function applyFilter($target, array $parameters, array $operators, ExecutionContext $context)
    {
        /** @var \Solarium\Client $target */

        /** @var string $searchQuery */
        $searchQuery = $this->execute($target, $operators, $parameters);

        $query = $target->createSelect();
        $query->createFilterQuery('rulerz')->setQuery($searchQuery);

        return $query;
    }
}
```

It uses the `execute()` method which is generated by the compiler and uses its
result on the Solr client (the target) to apply a filter.

Other traits would provide the implementation of the other methods.

## Registering the compilation target

This is the easiest part. Once your compilation target is implemented (and tested!),
you can add it to `\RulerZ\Rulerz`'s constructor to enable it:

```php
# no_execute
use rulerz\compiler\compiler;
use rulerz\compiler\target;
use rulerz\rulerz;

// compiler
$compiler = compiler::create();

// rulerz engine
$rulerz = new Rulerz(
    $compiler, [
        new Target\Native(),
        // …
        new \Acme\RulerZ\MyTarget(),
    ]
);
```

## That was it!

[Return to the index to explore the other possibilities of the library](index.md)
