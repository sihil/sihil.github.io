---
layout: post
title: First steps with Validated in cats
tags:
- Scala
- Functional Programming
- cats
---

This week I've taken my first steps with both the `Validated` and `NonEmptyList` types in cats. Also, in fact, my first steps with cats. I thought I'd share my experience as I had been putting off my first steps with cats and I shouldn't have been.

The context was trying to parse and resolve a new style of configuration from a YAML file in the Guardian's continuous delivery tool, [Riff-Raff][1].

I recommend reading the [herding cats][2] log of Eugene Yokota working through cats. I didn't find it particularly easy going (in part I think as I've neither used scalaz nor have a strong FP background) - however it is definitely worth reading for some good examples of using different concepts.

[1]: https://github.com/guardian/riff-raff
[2]: http://eed3si9n.com/herding-cats/index.html

## NonEmptyList

Let's start with `NonEmptyList` because it's the simpler of the two concepts. Super simple in fact. A `NonEmptyList` is a list that has at least one element. This makes it easier to reason about if an empty list is an illegal state. For the sake of brevity I frequently aliased `NonEmptyList` to `NEL`: `import cats.data.{NonEmptyList => NEL}`.

In my use case I'm parsing a file and at some stage have go from a normal `List` to a `NonEmptyList`. The main `NEL` constructor is `NEL(head, tail)`, but there is also `NEL.of(head)` if you have a singleton item and `NEL.fromList(list)` that returns an Option which can be None if the argument is an empty list (if you somehow know you have a list with at least one item you can use `NEL.fromListUnsafe(list)`). I'm reading a configuration though and am using `NEL` because an empty list is an error. Thus when using `NEL.fromList(list)` to go from the raw representation Some(nel) is the happy case and None is the unhappy case. This can work well with `Validated` as I'll mention later.
  
My main frustration with `NEL` is the disparity from `List`. `NEL` misses a heap of methods that I'm used to using. Some examples are `:::`, `mkString`, `distinct` and even `size`. This means adding a little `toList` boiler plate whenever you need to use one (or learning the alternative such as `concat` or `++` in the case of `:::`).

## Validated

I'll come back to `NEL`, but let's move on to look at `Validated`. The aim of this datatype is to represent either a success value or an error state. It's very similar to `Either` except that it is designed to accumulate errors rather than stop on the very first error. By returning multiple errors it should mean fewer fix and repeat cycles for a user in order to make a successful call. 

In my case I have a number of parsing and resolving phases. Each phase has to be completely correct before moving onto the next phase, but each phase can potentially have one or more errors. The phases are:

 - Parse the YAML
 - Resolve the YAML representation into an internal representation
 - Check that the internal representation makes sense (i.e. all the values are legal)
 
I initially modelled the errors as an `NEL` - using the convenient `ValidatedNel` type alias. I used a case class `ConfigError(content: String, message: String)` to store the errors and the type I returned was `ValidatedNel[ConfigError, <Success>]` (where <Success> was my successful type at that phase). That's equivalent to `Validated[NEL[ConfigError], <Success>]` and can be either `Invalid[NEL[ConfigError]]` or `Valid[<Success>]`.

In order for `Validated` to be able to accumulate errors easily the `Invalid` type is a `Semigroup` (a data type that is associative, i.e. (a + b) + c == a + (b + c) - both `List` and `NEL` are `Semigroup`s). This allows the methods of `Validated` to join the errors of multiple `Validated` instances together. In the end I switched from using `ValidatedNel[ConfigError, <Success>]` due to the amount of boilerplate that it needed (every time I created an error case I needed to make it into a `NEL` using `NEL.of` as well as instantiating `ConfigError`). I created another case class to hide this:

{% highlight scala %}
case class ConfigError(context: String, message: String)
case class ConfigErrors(errors: NEL[ConfigError])
object ConfigErrors {
  def apply(context: String, message: String): ConfigErrors = ConfigErrors(ConfigError(context, message))
  def apply(error: ConfigError): ConfigErrors = ConfigErrors(NEL.of(error))
  implicit val sg = new Semigroup[ConfigErrors] {
    def combine(x: ConfigErrors, y: ConfigErrors): ConfigErrors = ConfigErrors(x.errors.concat(y.errors))
  }
}
{% endhighlight %}

The `NEL` is still there, but I don't need to see it as I can directly instantiate `ConfigErrors("my context", "my error message")`. You'll notice that I've added a `Semigroup` implementation so that two `ConfigErrors` can be combined in the same way that `Validated` would have combined two `NEL` instances previously. So now my type is `Validated[ConfigErrors, <Success>]`.
 
### Parsing the YAML

The first phase was the parsing of the YAML. Slightly controversially I convert YAML into JSON and then use the Play JSON parser (because none of the YAML parsers in Scala are very good at the moment and I'm using a subset of YAML that can be expressed in JSON). Play JSON actually has a type that works pretty much exactly like `Validated`. It's not generic, but `JsResult` can hold either a successfully parsed value or a sequence of errors. It's quite simple to convert from one type to another: 

{% highlight scala %}
Json.fromJson[RiffRaffDeployConfig](json) match {
  case JsSuccess(config, _) => Valid(config)
  case JsError(errors :: tail) =>
    val nelErrors = NEL(errors, tail)
    Invalid(ConfigErrors(nelErrors.map{ case (path, validationErrors) =>
      ConfigError(s"Parsing $path", validationErrors.map(ve => ve.message).mkString(", "))
    }))
  case JsError(_) => ??? // not possible: if we match a JsError it will have at least one error
}
{% endhighlight %}
 
### Resolving the YAML representation into an internal representation

The output from the first phase is a set of case classes that match the structure of the input YAML. This next phase converts them into another case class after having checked that various things are true and done some expansions. Most of the input case class fields are `Option` but the output fields are not:

{% highlight scala %}
/** The two input case classes */
case class RiffRaffDeployConfig(
  stacks: Option[List[String]],
  regions: Option[List[String]],
  templates: Option[Map[String, DeploymentOrTemplate]],
  deployments: List[(String, DeploymentOrTemplate)]
)

case class DeploymentOrTemplate(
  `type`: Option[String],
  template: Option[String],
  stacks: Option[List[String]],
  regions: Option[List[String]],
  actions: Option[List[String]],
  app: Option[String],
  contentDirectory: Option[String],
  dependencies: Option[List[String]],
  parameters: Option[Map[String, JsValue]]
)

/** A deployment that has been parsed and validated out of a riff-raff.yml file. */
case class Deployment(
  name: String,
  `type`: String,
  stacks: NEL[String],
  regions: NEL[String],
  actions: Option[List[String]],
  app: String,
  contentDirectory: String,
  dependencies: List[String],
  parameters: Map[String, JsValue]
)
{% endhighlight %}

Instantiating a `Validated` is easy. `Validated.fromOption` is synonymous with `getOrElse` - it either becomes a `Valid(value)` if the option from `Some(value)` or an `Invalid` with the value you provide.
 
{% highlight scala %}
Validated.fromOption(
  templates.flatMap(_.get(parentTemplateName)),
  ConfigErrors(templateName, s"Template with name $parentTemplateName does not exist")
)
{% endhighlight %}

The interesting question is how you work with multiple `Validated` instances - either to combine them or chain them. There are a few options here. 

#### Combining using the cartesian operator 

If you have multiple `Validated` values that are combined to produce a single case class for example then you can do something like this:

{% highlight scala linenos %}
val typeField = Validated.fromOption(
  templated.`type`, 
  ConfigErrors(label, "No type field provided")
)
val stacksField = Validated.fromOption(
  templated.stacks.orElse(globalStacks).flatMap(NEL.fromList), 
  ConfigErrors(label, "No stacks provided")
)
val regionsField = Validated.fromOption(
  templated.regions.orElse(globalRegions).flatMap(NEL.fromList), 
  ConfigErrors(label, "No regions provided")
)
( typeField |@| stacksField |@| regionsField ) map { (deploymentType, stacks, regions) =>
  Deployment(
    name = label,
    `type` = deploymentType,
    stacks = stacks,
    regions = regions,
    actions = templated.actions,
    app = templated.app.getOrElse(label),
    contentDirectory = templated.contentDirectory.getOrElse(label),
    dependencies = templated.dependencies.getOrElse(Nil),
    parameters = templated.parameters.getOrElse(Map.empty)
  )
}
{% endhighlight %}

In this case we created three `Validated` fields (the first of type `String` and the other two of type `NEL[String]`), with their appropriate error messages. Then we use the cartesian operator `|@|` to join the three fields together. The result will either be Valid[(String, NEL[String], NEL[String])] (in which case we map over it and create our `Deployment` instance) or `Invalid` (the key thing is that the value in the `Invalid` object will contain all of the errors from the individual `Invalid` classes - added together using the `combine` operator of the `ConfigErrors` Semigroup instance).

Note also on lines 6 and 10 how `flatMap` and `NEL.fromList` are used together to produce the `Invalid` path if stacks/regions are either `None` or `Some(Nil)`.

#### Combining using the `combine` operator or `traverse`

If both the valid and invalid types of a `Validated` type have a `Semigroup` then the `Validated` type itself can be used as a `Semigroup`. That means that we can join together multiple `Validated` and it will accumulate either the errors or the success values. An example would be `Validated[ConfigErrors, NEL[Deployment]]`. `ConfigErrors` has a `Semigroup` instance as we defined it and `NEL` has one in cats. As a result we can take two instances of that type and `combine` them together. If two valid instances are combined together you'll get a valid with the two lists of `Deployment` concatenated together. Likewise if you have two invalid instances. In the case you have one of each, the invalid value will always win out and the valid value will be lost. You can do something like the following:

{% highlight scala %}
deployments.map { deployment =>
  DeploymentTypeResolver.validateDeploymentType(deployment, DeploymentType.all).map(List(_))
}.reduceLeft(_ combine _)
{% endhighlight %}

We map over a list of deployments and validate each one in turn. `validateDeploymentType` returns a `Validated` type which we map into a single element list (so we have a Semigroup) and then combine all of the results together. The result is a `Validated` that is either a list of all the successful results or a `ConfigErrors` of one or more errors.

A more elegant way of doing this is using the traverseU method:

{% highlight scala %}
import cats.syntax.traverse._
deployments.traverseU[ValidatedNel[ConfigError, Deployment]]{deployment =>
  DeploymentTypeResolver.validateDeploymentType(deployment, DeploymentType.all)
}
{% endhighlight %}

This has exactly the same result - we `traverse` the list and collect the successful results, unless a result isn't successful in which case we collect all of the errors.

#### Chain using `andThen`

In my case I have multiple phases and these phases are chained using the `andThen` method on `Validated`. Somewhat like `flatMap`, when `andThen` is called on an `Invalid` then it will fail fast and return itself. When called on a `Valid` however it calls the provided function with it's value and expects another `Validated` in return.
 
So to bring it all together we can do something like this:
 
{% highlight scala %}
val validatedDeployments = RiffRaffYamlReader.fromString(yamlConfig).andThen { config =>
  DeploymentResolver.resolve(config)
}.andThen { deployments =>
  deployments.traverseU[ValidatedNel[ConfigError, Deployment]]{deployment =>
    DeploymentTypeResolver.validateDeploymentType(deployment, DeploymentType.all)
  }
}
{% endhighlight %}

The cats library doesn't provide a `flatMap` for `Validated` as `andThen` doesn't match their definition of what `flatMap` does. This makes sense but sadly means that we can't use for comprehensions - making our code look uglier. However, given that we don't care too much about the purity of `flatMap` we can fix this! If we pimp a `flatMap` onto `Validated` like so:
 
{% highlight scala %}
implicit class RichValidated[E, A](validated: Validated[E, A]) {
  def flatMap[EE >: E, B](f: A => Validated[EE, B]): Validated[EE, B] = validated.andThen(f)
}
{% endhighlight %}

Then we can do this (which is far more readable):
 
{% highlight scala %}
for {
  config <- RiffRaffYamlReader.fromString(yamlConfig)
  deployments <- DeploymentResolver.resolve(config)
  validatedDeployments <- deployments.traverseU[ValidatedNel[ConfigError, Deployment]]{deployment =>
    DeploymentTypeResolver.validateDeploymentType(deployment, DeploymentType.all)
  }
} yield validatedDeployments
{% endhighlight %}

You can explore the code for yourself in the [`magenta-lib/src/main/scala/magenta/input` subtree of the Riff-Raff source code][3].

[3]: https://github.com/guardian/riff-raff/tree/master/magenta-lib/src/main/scala/magenta/input
