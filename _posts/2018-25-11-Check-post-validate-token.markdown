---
layout: post
title:  "Using NUnit to Ensure Post Methods Validate Anti-Forgery Tokens"
date:   2018-11-25 07:24:42 -0500
categories: [C#, MVC, NUnit, reflection]
tags: [C#, MVC, NUnit, reflection]
visible: 1
---

### Anti-Forgery Token Attributes

Microsoft's article [here](https://docs.microsoft.com/en-us/aspnet/web-api/overview/security/preventing-cross-site-request-forgery-csrf-attacks)
provides a great overview of preventing cross-site request forgery attacks with the use
of the built-in [```ValidateAntiForgeryTokenAttribute```](https://docs.microsoft.com/en-us/dotnet/api/system.web.mvc.validateantiforgerytokenattribute?view=aspnet-mvc-5.2).
One tool we can use to ensure that all of our controller methods decorated with the
[```HttpPostAttribute```](https://docs.microsoft.com/en-us/dotnet/api/system.web.mvc.httppostattribute?view=aspnet-mvc-5.2)
are also decorated with the ```ValidateAntiForgeryTokenAttribute```
is to write a unit test case.

#### Example Method

In our ```HomeController```, we have a method that is marked with a ```post``` verb:

``` csharp
public class HomeController : Controller
{
    [HttpPost, ValidateAntiForgeryToken]
    public ActionResult Create(PostViewModel postedModel)
    {
        // do something...
        return View();
    }
}
```

We want to ensure that every new ```post``` verb method is also decorated with the
```ValidateAntiForgeryToken``` attribute. We could accomplish this with an administrative
rule (coding guidelines) or rely on code reviews to catch when methods are not properly decorated,
or we can write one test that will check every ```post``` method in all controllers within an assembly.

#### The Unit Test

In our unit test project, we create a single test that ensures that a method is decorated
with the ```ValidateAntiForgeryToken``` attribute:

``` csharp
    [TestCaseSource(typeof(AllControllerPostMethods))]
    public void PostMethods_validate_antiforgery_token(MethodInfo method)
    {
        Assert.IsTrue(method
            .GetCustomAttributes(
                typeof(ValidateAntiForgeryTokenAttribute),
                false)
            .Any(),
            $"{method.Name} is not decorated with ValidateAntiForgeryToken.");
    }
```

The NUnit [```TestCaseSource```](https://github.com/nunit/docs/wiki/TestCaseSource-Attribute)
attribute directs the test runner to execute the test for each element in the enumeration. We
define our test data as every method that is decorated with ```HttpPost``` in all controllers
of the presentation layer assembly.

``` csharp
internal sealed class AllControllerPostMethods : IEnumerable
{
    public IEnumerator GetEnumerator()
    {
        var mvcAssembly = Assembly.Load("NameOfAssemblyContainingControllers");
        var controllerTypeInfos = mvcAssembly
            .DefinedTypes
            .Where(t => t.Name.EndsWith("Controller"));

        foreach (var typeInfo in controllerTypeInfos)
        {
            var methods = mvcAssembly
                .GetType(typeInfo.FullName)
                .GetRuntimeMethods()
                .Where(m => m.GetCustomAttributes(
                    typeof(HttpPostAttribute), false)
                    .Any());

            foreach (var method in methods)
            {
                yield return new object[]
                {
                    method
                };
            }
        }
    }
}
```

We load the assembly of the presentation layer under test so that we can get all types
that are controllers:

``` csharp
var mvcAssembly = Assembly.Load("NameOfAssemblyContainingControllers");
```

From this assembly, we can get the ```TypeInfo``` for all types whose names end with
 *Controller*. This works in the case that we have followed controller naming 
conventions when creating the classes.

``` csharp
var controllerTypeInfos = mvcAssembly
    .DefinedTypes
    .Where(t => t.Name.EndsWith("Controller"));
```

We then iterate over each ```TypeInfo``` representation of the controllers to get all
methods in the controller that are decorated with ```HttpPost```:

``` csharp
foreach (var typeInfo in controllerTypeInfos)
{
    var methods = mvcAssembly
        .GetType(typeInfo.FullName)
        .GetRuntimeMethods()
        .Where(m => m.GetCustomAttributes(
            typeof(HttpPostAttribute), false)
            .Any());

    foreach (var method in methods)
    {
        yield return new object[]
        {
            method
        };
    }
}
```

If this test fails, we know we have a post method that isn't configured to
validate anti-forgery tokens.
