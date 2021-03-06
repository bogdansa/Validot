<h1 align="center">
  <br />
    <img src="../assets/logo/validot-logo.svg" height="256px" width="256px" />
  <br />
  Validot
  <br />
</h1>


 <h3 align="center">Tiny lib for advanced model validation. With performance in mind.</h3>

  <br />
<p align="center">
  <a href="https://github.com/bartoszlenar/Validot/actions?query=branch%3Amaster+workflow%3ACI">
    <img src="https://img.shields.io/github/workflow/status/bartoszlenar/Validot/CI/master?style=for-the-badge&label=CI&logo=github&logoColor=white&logoWidth=20">
  </a>
  <a href="https://codecov.io/gh/bartoszlenar/Validot/branch/master">
    <img src="https://img.shields.io/codecov/c/gh/bartoszlenar/Validot/master?style=for-the-badge&logo=codecov&logoColor=white&logoWidth=20">
  </a>
  <a href="https://www.nuget.org/packages/Validot">
      <img src="https://img.shields.io/nuget/v/Validot?style=for-the-badge&logo=nuget&logoColor=white&logoWidth=20&label=STABLE%20VERSION">
  </a>
</p>
<p align="center">
  <a href="https://github.com/bartoszlenar/Validot/commits/master">
    <img src="https://img.shields.io/github/last-commit/bartoszlenar/Validot/master?style=flat-square">
  </a>
  <a href="https://github.com/bartoszlenar/Validot/releases">
    <img src="https://img.shields.io/github/release-date-pre/bartoszlenar/Validot?include_prereleases&style=flat-square&label=last%20release">
  </a>
  <a href="https://github.com/bartoszlenar/Validot/releases">
    <img src="https://img.shields.io/github/v/release/bartoszlenar/Validot?include_prereleases&style=flat-square&label=last%20release%20version">
  </a>
</p>

<div align="center">
  <h3>
    <a href="#quickstart">
      Quickstart
    </a>
    |
    <a href="#features">
      Features
    </a>
    |
    <a href="#project-info">
      Project info
    </a>
    |
    <a href="../docs/DOCUMENTATION.md">
      Documentation
    </a>
    |
    <a href="../docs/CHANGELOG.md">
      Changelog
    </a>
</div>

<p align="center">
    <a href="#validot-vs-fluentvalidation">
        🔥⚔️ Validot vs FluentValidation ⚔️🔥
    </a>
</p>
<br/>

<p align="center">
    Built with 🤘🏻by <a href="https://lenar.dev">Bartosz Lenar</a>
</p>

## Quickstart

Add the Validot nuget package to your project using dotnet CLI:

```
dotnet add package Validot
```

All the features are accessible after referencing single namespace:


``` csharp
using Validot;
```

And you're good to go! At first, create a specification for your model with the fluent api.

``` csharp
Specification<UserModel> specification = _ => _
    .Member(m => m.Email, m => m
        .Email().WithExtraCode("ERR_EMAIL")
        .MaxLength(100)
    )
    .Member(m => m.Name, m => m
        .Optional()
        .LengthBetween(8, 100)
        .Rule(name => name.All(char.IsLetterOrDigit)).WithMessage("Must contain only letter or digits")
    )
    .Rule(m => m.Age >= 18 || m.Name != null)
        .WithPath("Name")
        .WithMessage("Required for underaged user")
        .WithExtraCode("ERR_NAME");
```

The next step is to create a [validator](../docs/DOCUMENTATION.md#validator). As its name stands - it validates objects according to the [specification](../docs/DOCUMENTATION.md#specification). It's also thread-safe so you can seamlessly register it as a singleton in your DI container.

``` csharp
var validator = Validator.Factory.Create(specification);
```

Validate the object!

``` csharp
var model = new UserModel(email: "inv@lidv@lue", age: 14);

var result = validator.Validate(model);
```

The [result](../docs/DOCUMENTATION.md#result) object contains all information about the [errors](../docs/DOCUMENTATION.md#error-output). Without retriggering the validation process you can extract the desired form of an output.

``` csharp
result.AnyErrors; // bool flag:
// true

result.MessageMap["Email"] // collection of messages for "Email":
// [ "Must be a valid email address" ]

result.Codes; // collection of all the codes from the model:
// [ "ERR_EMAIL", "ERR_NAME" ]

result.ToString(); // compact printing of codes and messages:
// ERR_EMAIL, ERR_NAME
//
// Email: Must be a valid email address
// Name: Required for underaged user
```

* [See this example's real code](../tests/Validot.Tests.Functional/Readme/QuickStartFuncTests.cs)

## Features

### Advanced fluent API, inline

No more obligatory if-ology around input models or separate classes wrapping just validation logic. Write [specifications](../docs/DOCUMENTATION.md#specification) inline with simple, human-readable [fluent API](../docs/DOCUMENTATION.md#fluent0api). Native support for properties and fields, structs and classes, [nullables](../docs/DOCUMENTATION.md#asnullable), [collections](../docs/DOCUMENTATION.md#ascollection), [nested members](../docs/DOCUMENTATION.md#member) and all of the possible combinations.

``` csharp
Specification<string> nameSpecification = s => s
    .LengthBetween(5, 50)
    .SingleLine()
    .Rule(name => name.All(char.IsLetterOrDigit));

Specification<string> emailSpecification = s => s
    .Email()
    .Rule(email => email.All(char.IsLower)).WithMessage("Must contain only lower case characters");

Specification<UserModel> userSpecification = s => s
    .Member(m => m.Name, nameSpecification).WithMessage("Must comply with name rules")
    .Member(m => m.PrimaryEmail, emailSpecification)
    .Member(m => m.AlternativeEmails, m => m
        .Optional()
        .MaxCollectionSize(3).WithMessage("Must not contain more than 3 addresses")
        .AsCollection(emailSpecification)
    )
    .Rule(user => {

        return user.PrimaryEmail == null || user.AlternativeEmails?.Contains(user.PrimaryEmail) == false;

    }).WithMessage("Alternative emails must not contain the primary email address");
```

* [Guide through Validot's fluent API](../docs/DOCUMENTATION.md#fluent-api)
* [If you prefer approach of having separate class for just validation logic, it's also fully supported](../docs/DOCUMENTATION.md#specification-holder)

### Validators

Compact, highly optimized and thread-safe objects to handle the validation.

``` csharp
Specification<BookModel> bookSpecification = s => s
    .Optional()
    .Member(m => m.AuthorEmail, m => m.Optional().Email())
    .Member(m => m.Title, m => m.NotEmpty().LengthBetween(1, 100))
    .Member(m => m.Price, m => m.NonNegative());

var bookValidator =  Validator.Factory.Create(bookSpecification);

services.AddSingleton<IValidator<BookModel>>(bookValidator);
```

``` csharp
var bookModel = new BookModel() { AuthorEmail = "inv@lid_em@il", Price = 10 };

bookValidator.IsValid(bookModel);
// false

bookValidator.Validate(bookModel).ToString();
// AuthorEmail: Must be a valid email address
// Title: Required

bookValidator.Validate(bookModel, failFast: true).ToString();
// AuthorEmail: Must be a valid email address

bookValidator.Template.ToString(); // Template contains all of the possible errors:
// AuthorEmail: Must be a valid email address
// Title: Required
// Title: Must not be empty
// Title: Must be between 1 and 100 characters in length
// Price: Must not be negative
```

* [What Validator is and how it works](../docs/DOCUMENTATION.md#validator)
* [More about template and how to use it](../docs/DOCUMENTATION.md#template)

### Results

Whatever you want. [Error flag](../docs/DOCUMENTATION.md#anyerrors), compact [list of codes](../docs/DOCUMENTATION.md#codes), or detailed maps of [messages](../docs/DOCUMENTATION.md#messagemap) and [codes](../docs/DOCUMENTATION.md#codemap). With a sugar on top: friendly [ToString() printing](../docs/DOCUMENTATION.md#tostring) that contains everything, nicely formatted.


``` csharp
var validationResult = validator.Validate(signUpModel);

if (validationResult.AnyErrors)
{
    // check if a specific code has been recorded for Email property:
    if (validationResult.CodeMap["Email"].Contains("DOMAIN_BANNED"))
    {
        _actions.NotifyAboutDomainBanned(signUpModel.Email);
    }

    var errorsPrinting = validationResult.ToString();

    // save all messages and codes printing into the logs
    _logger.LogError("Errors in incoming SignUpModel: {errors}", errorsPrinting);

    // return all error codes to the frontend
    return new SignUpActionResult
    {
        Success = false,
        ErrorCodes = validationResult.Codes,
    };
}
```

* [Validation result types](../docs/DOCUMENTATION.md#result)

### Rules

Tons of [rules available out of the box](../docs/DOCUMENTATION.md#rules). Plus an easy way to [define your own](../docs/DOCUMENTATION.md#custom-rules) with full support of Validot internal features like [formattable message arguments](../docs/DOCUMENTATION.md#message-arguments).

``` csharp
public static IRuleOut<string> ExactLinesCount(this IRuleIn<string> @this, int count)
{
    return @this.RuleTemplate(
        value => value.Split(Environment.NewLine).Length == count,
        "Must contain exactly {count} lines",
        Arg.Number("count", count)
    );
}
```

``` csharp
.ExactLinesCount(4)
// Must contain exactly 4 lines

.ExactLinesCount(4).WithMessage("Required lines count: {count}")
// Required lines count: 4

.ExactLinesCount(4).WithMessage("Required lines count: {count|format=000.00|culture=pl-PL}")
// Required lines count: 004,00
```

* [List of built-in rules](../docs/DOCUMENTATION.md#rules)
* [Writing custom rules](../docs/DOCUMENTATION.md#custom-rules)
* [Message arguments](../docs/DOCUMENTATION.md#message-arguments)

### Translations

Pass errors directly to the end users in the language of your application.

``` csharp
Specification<UserModel> specification = s => s
    .Member(m => m.PrimaryEmail, m => m.Email())
    .Member(m => m.Name, m => m.LengthBetween(3, 50));

var validator =  Validator.Factory.Create(specification, settings => settings.WithPolishTranslation());

var model = new UserModel() { PrimaryEmail = "in@lid_em@il", Name = "X" };

var result = validator.Validate(model);

result.ToString();
// Email: Must be a valid email address
// Name: Must be between 3 and 50 characters in length

result.ToString(translationName: "Polish");
// Email: Musi być poprawnym adresem email
// Name: Musi być długości pomiędzy 3 a 50 znaków
```

* [How translations work](../docs/DOCUMENTATION.md#translations)
* [Custom translation](../docs/DOCUMENTATION.md#custom-translation)
* [How to selectively override built-in error messages](../docs/DOCUMENTATION.md#overriding-messages)

## Validot vs FluentValidation

A short statement to start with - [@JeremySkinner](https://twitter.com/JeremySkinner)'s [FluentValidation](https://fluentvalidation.net/) is a great piece of work and has been a huge inspiration for this project. True, you can call Validot a direct competitor, but it differs in some fundamental decisions and lot of attention has been focused on completely different aspects. If after reading this section you think you can bear another approach, api and [limitations](#fluentValidations-features-that-validot-is-missing), at least give Validot a try. You might be positively surprised. Otherwise, FluentValidation is a good, safe choice, as Validot is certainly less hackable and achieving some very specific goals might be either difficult or impossible.

### Validot is faster and consumes less memory

Before anything else; this document shows terribly oversimplified results of [BenchmarkDotNet](https://benchmarkdotnet.org/) execution, but the intention is to present the general trend only. To have truly reliable numbers, I highly encourage you to [run the benchmarks yourself](../docs/DOCUMENTATION.md#benchmarks).

There are three data sets, 10k models each; `ManyErrors` (every model has many errors), `HalfErrors` (around 60% have errors, the rest are valid), `NoErrors` (all are valid) and the rules reflect each other as much as technically possible. I did my best to make sure that the tests are just and adequate, but I'm a human being and I make mistakes. Really, if you spot errors [in the code](https://github.com/bartoszlenar/Validot/tree/master/tests/Validot.Benchmarks), framework usage, applied methodology... or if you can provide any counterexample proving that Validot struggles with some particular scenarios - I'd be very very very happy to accept a PR and/or discuss it on [GitHub Issues](https://github.com/bartoszlenar/Validot/issues).

To the point; the statement in the header is true, but it doesn't come for free. Wherever possible and justified, Validot chooses performance and less allocations over [flexibility and extra features](#fluentvalidations-features-that-validot-is-missing). Fine with that kind of trade-off? Good, because the validation process in Validot might be **~2.5x faster while consuming ~3.5x less memory**. Especially when it comes to memory consumption, Validot is usually far, far more better than that (depending on the use case it might be even **~15x more efficient** comparing to FluentValidation):

| Test | Data set | Library | Mean [ms] | Allocated [MB] |
| - | - | - | -: | -: |
| Validate | `ManyErrors` | FluentValidation | `747.66` | `686.80` |
| Validate | `ManyErrors` | Validot | `321.00` | `183.19` |
| FailFast | `ManyErrors` | FluentValidation | `748.11` | `686.80` |
| FailFast | `ManyErrors` | Validot | `14.20` | `31.9` |
| Validate | `HalfErrors` | FluentValidation | `658.10` | `684.60` |
| Validate | `HalfErrors` | Validot | `273.51` | `85.10` |
| FailFast | `HalfErrors` | FluentValidation | `666.12` | `684.60` |
| FailFast | `HalfErrors` | Validot | `185.19` | `64.96` |

* [Validate benchmark](../tests/Validot.Benchmarks/Comparisons/ValidationBenchmark.cs) - objects are validated.
* [FailFast benchmark](../tests/Validot.Benchmarks/Comparisons/ValidationBenchmark.cs) - objects are validated, the process stops on the first error.

FluentValidation's `IsValid` is a property that wraps a simple check whether the validation result contains errors or not. Validot has [AnyErrors](../docs/DOCUMENTATION.md#anyerrors) that acts the same way, but [IsValid](../docs/DOCUMENTATION.md#isvalid) is a dedicated special mode that doesn't care about anything else but the first rule predicate that fails. If the mission is only to verify the incoming model whether it complies with the rules (discarding all of the details), this approach proves to be better up to one order of magnitude:

| Test | Data set | Library | Mean [ms] | Allocated [MB] |
| - | - | - | -: | -: |
| IsValid | `ManyErrors` | FluentValidation | `750.33` | `686.80` |
| IsValid | `ManyErrors` | Validot | `14.43` | `31.21` |
| IsValid | `HalfErrors` | FluentValidation | `647.11` | `684.60` |
| IsValid | `HalfErrors` | Validot | `181.90` | `64.57` |
| IsValid | `NoErrors` | FluentValidation | `652.64` | `668.51` |
| IsValid | `NoErrors` | Validot | `266.63` | `78.82` |

* [IsValid benchmark](../tests/Validot.Benchmarks/Comparisons/ValidationBenchmark.cs) - objects are validated, but only to check if they are valid or not.

In fact, combining these two methods in most cases could be quite beneficial. At first [IsValid](../docs/DOCUMENTATION.md#isvalid) quickly verifies the object and if it contains errors - only then [Validate](../docs/DOCUMENTATION.md#validate) is executed to report the details. Of course in some extreme cases (megabyte-size data? millions of items in the collection? dozens of nested levels with loops in reference graphs?) traversing through the object twice could neglect the profit, but for the regular web api input validation it will certainly serve its purpose:

``` csharp
if (!validator.IsValid(model))
{
    errorMessages = validator.Validate(model).ToString();
}
```

| Test | Data set | Library | Mean [ms] | Allocated [MB] |
| - | - | - | -: | -: |
| Reporting | `ManyErrors` | FluentValidation | `753.50` | `721.01` |
| Reporting | `ManyErrors` | Validot | `419.60` | `335.99 ` |
| Reporting | `HalfErrors` | FluentValidation | `651.90` | `685.22` |
| Reporting | `HalfErrors` | Validot | `364.80` | `123.74` |

* [Reporting benchmark](../tests/Validot.Benchmarks/Comparisons/ReportingBenchmark.cs) benchmark:
  * FluentValidation validates model, and `ToString()` is called if errors are detected.
  * Validot processes the model twice - at first, with its special mode, [IsValid](../docs/DOCUMENTATION.md#isvalid). Secondly - in case of errors detected - with the standard method, gathering all errors and printing them with `ToString()`.

Benchmarks environment: Validot 1.0.0, FluentValidation 8.6.2, .NET Core 3.1.4, i7-9750H (2.60GHz, 1 CPU, 12 logical and 6 physical cores), X64 RyuJIT, macOS Catalina.

### Validot handles nulls on its own

In Validot, null is a special case [handled by the core engine](../docs/DOCUMENTATION.md#null-policy). You don't need to secure the validation logic from null as your predicate will never receive it.

``` csharp
Member(m => m.LastName, m => m
    .Rule(lastName => lastName.Length < 50) // 'lastName' is never null
    .Rule(lastName => lastName.All(char.IsLetter)) // 'lastName' is never null
)
```

### Validot treats null as error by default

All values are marked as required by default. In the above example, if `LastName` member were null, the validation process would exit `LastName` scope immediately only with this single error message:

```
LastName: Required
```

The content of the message is, of course, [customizable](../docs/DOCUMENTATION.md#withmessage)).

If null should be allowed, place [Optional](../docs/DOCUMENTATION.md#optional) command at the beginning:

``` csharp
Member(m => m.LastName, m => m
    .Optional()
    .Rule(lastName => lastName.Length < 50) // 'lastName' is never null
    .Rule(lastName => lastName.All(char.IsLetter)) // 'lastName' is never null
)
```

Again, no rule predicate is triggered. Also null `LastName` member doesn't result with errors.

* [Read more about how Validot handles nulls](../docs/DOCUMENTATION.md#null-policy)

### Validot's Validator is immutable

Once [validator](../docs/DOCUMENTATION.md#validator) instance is created, you can't modify its internal state or [settings](../docs/DOCUMENTATION.md#settings). If you need the process to fail fast (FluentValidation's `CascadeMode.StopOnFirstFailure`), use the flag:

``` csharp
validator.Validate(model, failFast: true);
```

### FluentValidation's features that Validot is missing

Features that might be in the scope and are technically possible to implement in the future:

* `await`/`async` support ([discuss it on GitHub Issues](https://github.com/bartoszlenar/Validot/issues/2))
* transforming values ([discuss it on GitHub Issues](https://github.com/bartoszlenar/Validot/issues/3))
* severities  ([discuss it on GitHub Issues](https://github.com/bartoszlenar/Validot/issues/4))
* failing fast only in a single scope ([discuss it on GitHub Issues](https://github.com/bartoszlenar/Validot/issues/5))
* validated value in the error message ([discuss it on GitHub Issues](https://github.com/bartoszlenar/Validot/issues/6))
* "smart paths" in the error message, e.g. `RootUserCollection` member becomes `Root User Collection` ([discuss it on GitHub Issues](https://github.com/bartoszlenar/Validot/issues/1))

Features that are very unlikely to be in the scope as they contradict with the project's principles, and/or would have very negative impact on performance, and/or are impossible to implement:

* Access to any stateful context in the rule condition predicate:
  * It implicates lack of support for dynamic message content and/or amount.
* Integration with ASP.NET or other frameworks:
  * Making such a thing wouldn't be a difficult task at all, but Validot tries to remain a single-purpose library and all integrations need to be done individually
* Callbacks:
  * Please react on [failure/success](../docs/DOCUMENTATION.md#anyerrors) after getting [validation result](../docs/DOCUMENTATION.md#result).
* Pre-validation:
  * All cases can be handled by additional validation and a proper if-else.
  * Also, the problem of root being null doesn't exist in Validot (it's a regular case, [covered entirely with fluent api](../docs/DOCUMENTATION.md#presence-commands))
* Rule sets
  * workaround; multiple [validators](../docs/DOCUMENTATION.md#validator) for different parts of the object.

## Project info

### Requirements

Validot is a dotnet class library targeting .NET Standard 2.0. There are no extra dependencies.

Please check the [official Microsoft document](https://github.com/dotnet/standard/blob/master/docs/versions/netstandard2.0.md) that lists all the platforms that can use it on.

### Versioning

[Semantic versioning](https://semver.org/) is being used very strictly. Major version is updated only when there is a breaking change, no matter how small it might be (e.g. adding extra function to the public interface). On the other hand, huge pack of new features will bump the minor version only.

Before every major version update, at least one preview version is published.

### Reliability

Unit tests coverage hits 100% very close, it can be detaily verified on [codecov.io](https://codecov.io/gh/bartoszlenar/Validot/branch/master).

Before publishing, each release is tested on the [latest versions](https://help.github.com/en/actions/reference/virtual-environments-for-github-hosted-runners#supported-runners-and-hardware-resources) of operating systems:

* macOS
* Ubuntu
* Windows Server

using the [LTS versions](https://dotnet.microsoft.com/platform/support/policy/dotnet-core) of the underlying frameworks:

* .NET Core 3.1
* .NET Core 2.1
* .NET Framework 4.8 (Windows only)

### Performance

Benchmarks exist in the form of  [the console app project](https://github.com/bartoszlenar/Validot/tree/master/tests/Validot.Benchmarks) based on [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet). Also, you can trigger performance tests [from the build script](../docs/DOCUMENTATION.md#benchmarks).

### Documentation

The documentation is hosted alongside the source code, in the git repository, as a single markdown file: [DOCUMENTATION.md](./../docs/DOCUMENTATION.md).

Code examples from the documentation live as [functional tests](https://github.com/bartoszlenar/Validot/tree/master/tests/Validot.Tests.Functional).

### Development

The entire project ([source code](https://github.com/bartoszlenar/Validot), [issue tracker](https://github.com/bartoszlenar/Validot/issues), [documentation](./../docs/DOCUMENTATION.md) and [CI workflows](https://github.com/bartoszlenar/Validot/actions)) is hosted here on github.com.

Any contribution is more than welcome. If you'd like to help, please don't forget to check out the [CONTRIBUTING](./../docs/CONTRIBUTING.md) file and [issues page](https://github.com/bartoszlenar/Validot/issues).

### Licencing

Validot uses [MIT licence](../LICENSE). Long story short; you are more than welcome to use it anywhere you like, completely free of charge and without oppressive obligations.