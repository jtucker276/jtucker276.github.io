---
layout: post
title:  "Dependency Inversion in Custom MVC Authorization Attributes"
date:   2018-11-23 10:42:42 -0500
categories: [C#, MVC, Ninject, Dependency Inversion, IoC]
tags: [C#, MVC, Ninject, Dependency Inversion, IoC]
visible: 1
---

* TOC
{:toc}

### The Challenge

Given that we have a .NET C# MVC application that
does not leverage an implementation of a ```RoleProvider```
 (see [here](https://docs.microsoft.com/en-us/dotnet/api/system.web.security.roleprovider?view=netframework-4.7.2))
and relies on a persistent store for role assignments, how do we keep controller actions
restricted to only authorized users?

### Authorization Attributes

The ```System.Web.MVC.AuthorizeAttribute``` can generally be used when using
out-of-the-box role providers to decorate controllers or action methods to provide the
roles, users, etc. allowed to access the respective class and/or method:

``` csharp
[Authorize(Roles = "View")]
public class HomeController : Controller
{
    [Authorize(Roles = "Admin")]
    public ActionResult Index()
    {
        return View();
    }
}
```

We can write custom implementations of the ```AuthorizeAttribute``` to perform
our own business logic, or to access a persistent store for checking user role assignments.

### Dependency Inversion, Injection, and Resolution

#### Dependencies

Our ```HomeController``` depends on an ```IMessageService``` to provide some
message to the ```Index``` view -- contrived, maybe, but this extends to other
services as necessary.

``` csharp
public class HomeController : Controller
{
    private readonly IMessageService _messageService;

    public HomeController(IMessageService messageService)
    {
        _messageService = messageService;
    }

    public ActionResult Index()
    {
        ViewBag.InjectedMessage = _messageService.GetMessage();

        return View();
    }
}
```

Our ```IMessageService``` interface definition:

``` csharp
public interface IMessageService
{
    string GetMessage();
}
```

And our stub concretion:

``` csharp
public class MessageService : IMessageService
{
    public string GetMessage()
    {
        return "This is a stub message.";
    }
}
```


Similarly, our custom authorization attribute will depend on an ```IDependentService```
to provide an answer to whether or not an action is allowable:

``` csharp
public interface IDependentService
{
    bool IsAuthorized();
}
```

And it's stub concretion:

``` csharp
public class StubService : IDependentService
{
    public bool IsAuthorized()
    {
        return true;
    }
}
```

The less trivial implementation of ```IDependentService``` would itself be dependent
on a persistent store to look up user role memberships, perform other business logic, etc.
But for now, it's enough to just get a service injected into the attribute.


#### Injection

In this particular case, we are handling dependency inversion within the application
(primarily by parameterized constructor injection) with Ninject.

Within our ```Global.asax```, we derive from ```NinjectHttpApplication``` rather
than ```System.Web.HttpApplication```:

``` csharp
public class MvcApplication : NinjectHttpApplication
{
    // This is the standard Application_Start method
    // of the Global.asax, but renamed to not hide
    // the NinjectHttpApplication Application_Start method.
    private void Application_Start_Base()
    {
        AreaRegistration.RegisterAllAreas();
        GlobalConfiguration.Configure(WebApiConfig.Register);
        FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
        RouteConfig.RegisterRoutes(RouteTable.Routes);
        BundleConfig.RegisterBundles(BundleTable.Bundles);
    }

    protected override void OnApplicationStarted()
    {
        base.OnApplicationStarted();
        Application_Start_Base();
    }

    protected override IKernel CreateKernel()
    {
        var kernel = new StandardKernel();

        kernel.Load(new PresentationModule());

        return kernel;
    }
}
```


#### Binding

We bind concretions within modules to avoid having a brobdingnagian set of
bindings in the future. (See [this](https://github.com/ninject/Ninject/wiki/Modules-and-the-Kernel)
Ninject wiki on modules.) For the presentation layer, we have a single ```PresentationModule```
that derives from ```NinjectModule```, which can be broken apart 
later into smaller modules if necessary.

``` csharp
public class PresentationModule : NinjectModule
{
    public override void Load()
    {
        Bind<IMessageService>()
            .To<MessageService>();

        Bind<IDependentService>()
            .To<StubService>();
    }
}
```

#### Resolution

When we implement our custom ```AuthorizeAttribute```, it depends on the ```IDependentService```
to provide the yes/no answer to our authorization question. Note that we are deriving
from ```System.Web.Mvc.AuthorizeAttribute``` and not
```System.Web.Http.AuthorizeAttribute``` which is used in WebAPI.

It would be great to be able to use parameterized constructor injection to resolve
the dependency on ```IDependentService```:

``` csharp
public class CustomAuthorizeAttribute : AuthorizeAttribute
{
    private IDependentService _someService;

    public CustomAuthorizeAttribute(IDependentService dependentService)
    {
        _someService = dependentService;
    }

    protected override bool AuthorizeCore(HttpContextBase httpContext)
    {
        var systemDecision = base.AuthorizeCore(httpContext);
        var serviceDecision = _someService.IsAuthorized();
        return true;
    }
}
```

...but we then get into invalid attribute parameter types, the inability to pass
an argument when we use the attribute as a decoration, and we can't compile the project.

We do, however, have access to the static ```DependencyResolver``` to help us
resolve the dependency (see [this](https://docs.microsoft.com/en-us/dotnet/api/system.web.mvc.dependencyresolver?view=aspnet-mvc-5.2)
Microsoft documentation). This does mean that we're using the service locator anti-pattern,
but what other options do we really have?

``` csharp
public class CustomAuthorizeAttribute : AuthorizeAttribute
{
    protected IDependentService SomeService => 
        DependencyResolver.Current.GetService<IDependentService>();

    protected override bool AuthorizeCore(HttpContextBase httpContext)
    {
        var systemDecision = base.AuthorizeCore(httpContext);
        var serviceDecision = SomeService.IsAuthorized();
        return true;
    }
}
```

Alternatively, we could decorate the ```IDependentService``` property with Ninject's
```Inject``` deocration to tell Ninject to resolve the property, but this tightly couples
our attribute to a specific IoC concretion. If we swap out from Ninject in the future,
this is just another place that we have to update.

``` csharp
[Inject]
protected IDependentService SomeService { get; set; }
```

### Conclusion

By using an authorization attribute to limit access to controllers and methods,
we are keeping with .Net MVC conventions -- while at the same time we are straying
from convention by using our own persistent role storage outside the context of
ASP.Net roles. 

We build our custom authorization attribute to depend on a business service to provide
additional authorization information. While parameterized constructor injection is a
feasible implementation of dependency resolution in controllers and some other objects, we
are forced to search out another solution for objects deriving from ```AuthorizeAttribute```.

We do want to avoid using the service locator "anti-pattern" when we have other options
like constructor injection that more clearly indicate object dependencies. However, since that
is not an option in some cases, the "anti-pattern" can become a valid pattern assuming its use
is limited and well documented.