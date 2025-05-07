+++
title = "Tips on migrating legacy .NET Framework projects to modern .NET"
date = "2025-05-07"
tags = [["dotnet"]]
description = "Various tips on migrating .NET Framework 4.x applications to modern .NET versions."
+++

Last week, a friend pointed me to the
[.NET Conf Focus on Modernization][dotnet-conf] that happened a few weeks ago.
The sessions (now archived on
[YouTube](https://www.youtube.com/watch?v=ZwmD__W8UFM&list=PLdo4fOcmZ0oXmKFtZ7ca3uHjJl3apIsuO))
inspired me to share some of my tips that I've learned while migrating many
legacy projects from .NET Framework to modern .NET.

[dotnet-conf]: https://focus.dotnetconf.net/

> Much of this code was copied or inspired from other members of the .NET
> community online. I have posted attribution of the original sources I have
> stored in my notes, but for some of these tips, I no longer remember if it
> was from my research or from reading others' work. If I have accidentally
> plagiarized anyone, [let me know](https://fosstodon.org/@iinuwa)!

> Also note that, I refer the top level class for the web
> app defined in `Global.asax.cs` as `Global`. Some applications have named this class
> `WebApplication`, depending on which template was used to generate the
> application, so you can change the name wherever that appears.

## Table of Contents

- [Update to .NET Framework 4.8](#update-to-net-framework-4-8)
- [Migrate configuration to IConfiguration](#migrate-configuration-to-iconfiguration)
- [Update WebClient to HttpClient](#update-webclient-to-httpclient)
- [Migrate to use Dependency Injection](#migrate-to-use-dependency-injection)
- [Consider using System.Text.Json for serialization](#consider-using-system-text-json-for-serialization)
- [Add Tracing](#add-tracing)

## Update to .NET Framework 4.8

### Why upgrade to .NET Framework 4.8?

Updating to the latest version of .NET Framework (4.8, or 4.8.1 for Windows
Server 2022+) is a useful step, though not required. There aren't many
significant API changes between 4.8 and 4.7, though if you're coming from 4.6,
you may be missing out on some cryptography enhancements, as well as HTTP/2 and
more secure default TLS negotiation. The .NET 4.8 runtime comes with
performance improvements over previous versions as well. Finally, older
frameworks will eventually be unsupported by Microsoft. For example, according
to the .NET Framework lifecycle policy, 4.7.1 will be EOL by the beginning of
2027.

The .NET Framework 4.x versions are also backward compatible, so dependencies
should all still work after an upgrade. There's little reason not to upgrade if
it's available for your operating system.

### How to upgrade to .NET Framework 4.8

Visual Studio will prompt you to upgrade the runtime of a project if you don't
have the project's current configured runtime installed. I've found this
automatic upgrade method to vary in effectiveness between projects. Here are
also some manual steps in case the automated method doesn't work.

1. Unload the project in Visual Studio to open its `.csproj` file. (Visual Studio
   doesn’t let you edit legacy .NET Framework project files directly unless you
   unload it first.) Alternatively, navigate to the project directly in File
   Explorer and open the `.csproj` file in a text editor like Notepad.

2. In the `.csproj` file, set the `TargetFrameworkVersion` to `v4.8` (note the
   `v` prefix) and set the `AutoGenerateBindingRedirects` property to `true`:

   ```xml
   <Project>
   <!-- ... -->
     <TargetFrameworkVersion>v4.8</TargetFrameworkVersion>

     <!-- Edit or add this property if not present anywhere else in the file -->
     <PropertyGroup>
       <AutoGenerateBindingRedirects>true</AutoGenerateBindingRedirects>
     </PropertyGroup>
     <!-- All the other project stuff... -->
   ```

3. For web applications, edit the `Web.config` file. Under the `<system.web>` section, make sure the
   `<compilation>` and `<httpRuntime>` elements have the
   `targetFramework` set to `4.8` (no `v` prefix here).

   ```xml
   <system.web>
     <!-- ... -->
     <httpRuntime targetFramework="4.8" maxRequestLength="10240" />
     <compilation targetFramework="4.8" debug="true">
       <!-- ... -->
     </compilation>
   </system.web>
   ```

4. Also in the `Web.config` file (or `app.config` for console applications), in
   the `<configuration>` section, delete everything inside the `<runtime>`
   element (which should contain `<dependentAssembly>` elements), but leave the
   `<runtime>` element itself:

   ```xml
   <configuration>
     <runtime />
   </configuration>
   ```

5. Save both files, and then open the project in Visual Studio and attempt to
   build the project. You will get build warnings about assembly
   binding redirects like this:

   > Found conflicts between different versions of the same dependent assembly.
   > Please add the following binding redirects to the "runtime" node in your
   > application configuration file.

6. Double-click on one of the warnings to make Visual Studio
   auto-generate the new binding redirects for you, and then
   build again.

7. Test your application to make sure no errors related to assembly bindings
   occur. Sometimes the automatic generation of bindings results in duplicate
   bindings or some missing bindings. You may need to remove and re-add SDK
   references or NuGet packages in order for Visual Studio to work those out
   properly.

8. If all goes well, congrats! You’ve updated to .NET Framework 4.8.

## Migrate configuration to IConfiguration

Whether you're building a web or console application, `IConfiguration` is the
de facto standard interface used in modern .NET to store and read configuration
values. The Microsoft implementation, `Microsoft.Extensions.Configuration`
comes with many default configuration sources, like JSON files, environment
variables, command line arguments and more. You can also create your own custom
configuration sources, for example, if you wanted to store values in a
database.

### Why use IConfiguration?

Updating your application to begin using the `IConfiguration` pattern can make
it easier to upgrade to .NET, as it is the stepping stone to the `Startup.cs` app
configuration pattern and dependency injection, which ASP.NET Core uses
heavily. The `Microsoft.Extensions.Configuration` package targets
`netstandard2.0`, so it is available for use in .NET Framework projects.

### Bootstrapping IConfiguration

First, add `Microsoft.Extensions.Configuration` package to your project.

Then, build the configuration on app startup and make it globally accessible to
your application.

```cs
// Global.asax.cs

using Microsoft.Extensions.Configuration;

// Make ConfigurationManager explicitly reference the .NET Framework
ConfiguratioManager class to remove ambiguiity.
using ConfigurationManager = System.Configuration.ConfigurationManager;

public class Global : HttpApplication // This class may also be called MvcApplication
                                      // depending on the template used to create this project.
{
    public static IConfiguration Configuration; // Add this property

    // ...

    void Application_Start(object sender, EventArgs e)
    {
        // ...
        // Add the following lines
        Configuration = new ConfigurationBuilder()
            .AddJsonFile("appsettings.json", optional: false)
            .AddInMemoryCollection(new Dictionary<string, string>
            {
                ["Foo"] = "bar",
            })
            .Build();
    }
}
```

Create an `appsettings.json` in the root of your project with an empty JSON
object `{}`, and ensure that the file is copied to the publish directory using
the "Preserve Newest" option by either editing the file properties in the
Visual Studio Solution Explorer, or by editing the relevant line in the
`.csproj`:

```xml
<Content Include="appsettings.json">
  <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
</Content>
```

Once you have the `IConfiguration` object set up to read in your application,
you will need to update the references.

### Update configuration references

You can choose to migrate the keys from `Web.config` to `appsettings.json` now,
and update your application to read from `Global.Configuration` instead of
`ConfigurationManager`.

You can use `IConfiguration[string key]` syntax for retrieving config values as
strings, or use `IConfiguration.GetValue<T>`, which will convert the string
value into the type you want.

I recommend storing a reference to the `Global.Configuration` object within
your class as a `static readonly` field. It is a bit tedious, but removing
globals will make it easier to transition to use dependency injection later.

```diff
  class FooService
  {
+     private static readonly IConfiguration _config = Global.Configuration;

#     // ...

      private void DoFoo()
      {
-         Uri baseUrl = ConfigurationManager.AppSettings["FooApiBaseKey"]
+         Uri baseUrl = _config.GetValue<Uri>["FooApiBaseKey"];
#         // ...
-         string filepath = ConfigurationManager.AppSettings["BarFile"];
+         string filepath = _config["BarFile"];
      }
```


### Incrementally replace IConfiguration

I have also found it helpful for larger applications to defer the migration of
the configuration keys to a separate step, and introduce a shim that imports
keys from `ConfigurationManager` into the `IConfiguration` instance
automatically. This incremental approach allows you to update the configuration
references class-by-class rather than key-by-key, which can result in larger
commits if the configuration key is used in many places.

Here is an example configuration provider that you can build on:

```cs
// App_Start/LegacyAppSettingsConfigurationSource.cs
using Microsoft.Extensions.Configuration;

namespace FooNamespace
{
    public class LegacyAppSettingsConfigurationSource : IConfigurationSource
    {
        public IConfigurationProvider Build(IConfigurationBuilder builder)
            => new LegacyAppSettingsConfigurationProvider();
    }
}
```

```cs
// App_Start/LegacyAppSettingsConfigurationProvider.cs
using System;
using System.Linq;
using Microsoft.Extensions.Configuration;
using ConfigurationManager = System.Configuration.ConfigurationManager;

namespace FooNamespace
{
    public class LegacyAppSettingsConfigurationProvider : ConfigurationProvider
    {
        public override void Load()
        {
            // These keys are used in ASP.NET MVC runtime, but not in ASP.NET
            Core, so let's exclude them so we know not to migrate them later.
            string[] excludedKeys = new string[]
            {
                "enableSimpleMembership",
                "webpages:Version",
                "webpages:Enabled",
                "PreserveLoginUrl",
                "ClientValidationEnabled",
                "UnobtrusiveJavaScriptEnabled",
            };
            Data = ConfigurationManager.AppSettings.AllKeys
                .Where(key => !excludedKeys.Contains(key))
                .ToDictionary(key => key, key => ConfigurationManager.AppSettings[key]);
        }
    }
}
```

```cs
// App_Start/LegacyAppSettingsConfigurationExtensions.cs
using Microsoft.Extensions.Configuration;

namespace FooNamespace
{
    public static class LegacyAppSettingsConfigurationExtensions
    {
        public static IConfigurationBuilder AddLegacyAppSettings(this IConfigurationBuilder builder)
        {
            builder.Add(new LegacyAppSettingsConfigurationSource());
            return builder;
        }
    }
}
```

Then you can add the keys from `ConfigurationManager` before your
`appsettings.json`, so that `appsettings.json` takes precedence over
`Web.config`:

```cs
// Global.asax.cs

Configuration = new ConfigurationBuilder()
  .AddLegacyAppSettings()
  .AddJsonFile("appsettings.json", optional: false)
  .Build();
```

Either way you choose, you will wind up with an application that supports `IConfiguration`.


## Update WebClient to HttpClient

In modern .NET, HTTP requests are handled using the `HttpClient` class, while
.NET Framework uses `WebClient` (and its related classes, like `WebRequest`).

This is an invasive change, mostly because it involves going from synchronous
HTTP calls to asynchronous ones. Depending on how you've constructed your
application, this may take a long time to migrate to using async/await.

### Why update to HttpClient?

If this is so invasive, why should we do this?

Since `WebClient` is supported in modern .NET, this step can be done before or
after migration. However, the support in .NET is only intended for migration
purposes and may have significant performance impacts, so I recommend to begin
working on it before migration.

Besides that, `HttpClient` is instrumented with `Activity` instances, which are
useful setting up distributed traces. (See below for more details about
distributed tracing in .NET Framework.)  TODO: link to section below

Some tips:

### LINQ and async/await

If you are using WebClient's synchronous methods in LINQ methods, you may need
to refactor those calls to make them function properly with async code.

The easiest thing to do is to split the LINQ methods into separate methods for
each async operation, using [`Task.WhenAll()`][task-whenall] to `await` on all
the web requests simultaneously, then continue processing the responses.

```cs
string[] ids = { "abc", "def" };
IEnumerable<string> usernames = ids.Select(id => GetFoo(id).Username);

private Foo GetFoo(string id)
{
    HttpWebResponse response = HttpWebRequest.Create($"{_baseUrl}/{id}").GetResponse();
    MemoryStream ms = new MemoryStream();
    response.GetResponseStream().CopyTo(ms);
    byte[] body = ms.ToArray();
    return DeserializeFoo(body);
}
```

```cs
string[] ids = { "abc", "def" };
IEnumerable<string> foos = await Task.WhenAll(ids.Select(id => GetFoo(id));
IEnumerable<string> usernames = foos.Select(foo => foo.Username);

private async Task<Foo> GetFoo(string id)
{
    HttpResponseMessage response = await _httpClient.GetAsync(${"_baseUrl}/{id}");
    byte[] body = await response.Content.ReadAsByteArrayAsync()
    return DeserializeFoo(body);
}
```

[task-whenall]: https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.whenall?view=netframework-4.8.1

### Avoid the System.Net.Http NuGet package

.NET Framework 4.8 comes with version 4.2.0.0 of the `System.Net.Http`
assembly, but this is not the latest version. You could import the latest
available version from NuGet, 4.3.4, but this causes problems with binding
redirects when used in ASP.NET. Microsoft has also stopped releasing patches
for the NuGet package, but .NET Framework runtimes will still receive security
patches as long as they are supported.

### Use a single HttpClient instance per class

In every class where HTTP requests are sent, use a `static readonly
HttpClient` property to reference the client. This avoids [resource
exhaustion](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines#recommended-use)
due to shared TCP sockets. Using a property within each class (rather than a
single, global `HttpClient`) makes it easy to transition nicely to using
dependency injection for `HttpClient`s, where client lifetime will be managed
automatically.

```cs
class FooService
{
    private static readonly HttpClient _httpClient = new HttpClient()
    {
        // Since Global.Configuration is also static, we can also reference it here.
        BaseAddress = new Uri(Global.Configuration["FooApiBaseUrl"])
    };

    // ...


    async Task<Foo> GetFoo(string id)
    {
        // GET /foo/123
        HttpResponseMessage response = await _httpClient.GetAsync(id);
        // ...
    }
}
```

## Migrate to use Dependency Injection

### What is dependency injection?

_Dependency injection_ is a pattern that allows objects to declare what
dependencies they need, and require the caller of the object to provide the
dependencies. This can make the code for each individual object more focused,
as it does not need to be concerned with how to create the dependencies it
needs.

`Microsoft.Extensions.DependencyInjection` is the dependency injection (DI)
system used in ASP.NET Core by default, as well as modern .NET console
applications. It is also colloquially referred to as "Microsoft DI" to
distinguish from some other DI systems, like Autofac and Ninject.

For more information on dependency injection in .NET, read
[Microsoft's docs][di-docs] on it.


### Why use dependency injection in ASP.NET?

Dependency injection was not integrated with ASP.NET from the start; instead,
ASP.NET uses many global or thread-local static fields to access data. On the
other hand, ASP.NET Core integrates very tightly with DI frameworks. If you are
not using dependency injection for your services already, then you can add
Microsoft DI to your ASP.NET application to make the migration to ASP.NET Core
easier.

Microsoft DI is especially useful for creating HTTP client classes with managed
`HttpClient` lifetimes through the `IHttpClientFactory` interface. Using DI,
may also help you to more explicitly define the relationships between your
services, which may help you find where there may be unnecessary coupling in
your application's architecture.

### How to add dependency injection to ASP.NET

Before doing this, you should at least set up `IConfiguration` as mentioned the
[previous section](#migrate-configuration-to-iconfiguration).

> For ease of implementation, this guides implements
> the Startup.cs pattern from ASP.NET Core 5 and lower rather than the minimal
> hosting model/builder pattern from ASP.NET Core 6+ for configuring services.
> After fully migrating to ASP.NET Core, you can use Microsoft's
> [migration guide][startup-migration-guide] to convert to the minimal hosting
> model so that your application aligns with modern code templates and samples.

[startup-migration-guide]: https://learn.microsoft.com/en-us/aspnet/core/migration/50-to-60?view=aspnetcore-9.0&tabs=visual-studio#new-hosting-model

This pattern is based on [David Fowl's very helpful gist on GitHub](https://gist.github.com/davidfowl/563a602936426a18f67cd77088574e61).

1. First, add latest versions (9.0.0 at time of writing) of
   `Microsoft.Extensions.DependencyInjection` and `Microsoft.Extensions.Http`
   packages to your project.
2. Add a new file, `Startup.cs` in the root of your project.

   ```cs
   // Startup.cs

   using Microsoft.Extensions.Configuration;
   using Microsoft.Extensions.DependencyInjection;

   namespace FooNamespace
   {
       public class Startup
       {
         public Startup(IConfiguration configuration)
         {
             Configuration = configuration;
         }

         public IConfiguration Configuration { get; }

         public void ConfigureServices(IServiceCollection services)
         {
         }
     }
   }
   ```

3. Implement the ASP.NET dependency resolver interface using Microsoft DI:

   ```cs
   // App_Start/ServiceProviderDependencyResolver.cs
   using System;
   using System.Collections.Generic;
   using System.Web;
   using System.Web.Http.Dependencies;
   using System.Web.Mvc;
   using Microsoft.Extensions.DependencyInjection;

   namespace FooNamespace
   {
       internal class ServiceProviderDependencyResolver :
           System.Web.Mvc.IDependencyResolver,
           System.Web.Http.Dependencies.IDependencyResolver
       {
           private IServiceScope _serviceScope;
           private readonly IServiceProvider _serviceProvider;
           public ServiceProviderDependencyResolver(IServiceProvider serviceProvider)
           {
               _serviceProvider = serviceProvider;
           }

           public IDependencyScope BeginScope()
           {
               _serviceScope = _serviceProvider.CreateScope();
               return new ServiceProviderDependencyResolver(_serviceScope.ServiceProvider);
           }

           public void Dispose()
           {
               _serviceScope?.Dispose();
           }

           public object GetService(Type serviceType)
           {
               if (HttpContext.Current?.Items[typeof(IServiceScope)] is IServiceScope scope)
               {
                   return scope.ServiceProvider.GetService(serviceType);
               }
               return _serviceProvider.GetService(serviceType);
           }

           public IEnumerable<object> GetServices(Type serviceType)
           {
               if (HttpContext.Current?.Items[typeof(IServiceScope)] is IServiceScope scope)
               {
                   return scope.ServiceProvider.GetServices(serviceType);
               }
               return _serviceProvider.GetServices(serviceType);
           }
       }
   }
   ```

4. Create the custom dependency resolver in `Global.asax.cs`:

   ```diff
   // Global.asax.cs
   + using Microsoft.Extensions.DependencyInjection;
     namespace FooNamespace
     {
         public class Global : HttpApplication
         {
   +         public static IServiceProvider ServiceProvider { get; private set; }
   #         // ...
             protected void Application_Start()
             {
                 Configuration = new ConfigurationBuilder()
                     .AddLegacyAppSettings()
                     .AddJsonFile("appsettings.json", optional: false)
                     .Build();
   +             ServiceProvider = CreateServiceProvider(Configuration);
   #             // ...
              }
   +
   +         private static IServiceProvider CreateServiceProvider(IConfiguration configuration)
   +         {
   +             var services = new ServiceCollection();
   +
   +             // automatically register Controller classes, like ASP.NET Core does.
   +             IEnumerable<Type> controllers = typeof(Startup).Assembly.GetExportedTypes()
   +                 .Where(t => !t.IsAbstract && !t.IsGenericTypeDefinition)
   +                 .Where(t => typeof(IHttpController).IsAssignableFrom(t)
   +                 || t.Name.EndsWith("Controller", StringComparison.OrdinalIgnoreCase));
   +             services.AddTransient((_) => configuration);
   +             foreach (Type type in controllers)
   +             {
   +                 services.AddTransient(type);
   +             }
   +
   +             var startup = new Startup(configuration);
   +             startup.ConfigureServices(services);
   +
   +             ServiceProvider serviceProvider = services.BuildServiceProvider();
   +
   +             return serviceProvider;
   +         }
   ```

5. Create an IIS module for the custom dependency resolver:

   ```cs
   // App_Start/ServiceScopeModule.cs
   using System;
   using System.Web;
   using System.Web.Mvc;
   using Microsoft.Extensions.DependencyInjection;

   namespace MBI.DonorFlow.App
   {
       internal class ServiceScopeModule : IHttpModule
       {
           private static IServiceProvider _serviceProvider;

           public void Dispose()
           {

           }

           public void Init(HttpApplication context)
           {
               context.BeginRequest += Context_BeginRequest;
               context.EndRequest += Context_EndRequest;
           }

           private void Context_EndRequest(object sender, EventArgs e)
           {
               var context = ((HttpApplication)sender).Context;
               if (context.Items[typeof(IServiceScope)] is IServiceScope scope)
               {
                   scope.Dispose();
               }
           }

           private void Context_BeginRequest(object sender, EventArgs e)
           {
               var context = ((HttpApplication)sender).Context;
               context.Items[typeof(IServiceScope)] = _serviceProvider.CreateScope();
           }

           public static void SetServiceProvider(IServiceProvider serviceProvider)
           {
               _serviceProvider = serviceProvider;
           }
       }
   }
   ```

6. Register the module in `Global.asax.cs`:

   ```diff
   + [assembly: PreApplicationStartMethod(typeof(Global), "InitModule")]
     namespace FooNamespace
     {
         public class Global : HttpApplication
         {
   #         // ...
   +         public static void InitModule()
   +         {
   +             RegisterModule(typeof(ServiceScopeModule));
   +         }
         }
     }
   ```

7. Register the DI Service Provider. Your project may use either or both Web API
   controllers (for API endpoints) and MVC controllers (for views); configure
   the ones you need.

   ```diff
   + using System;
     using System.Web.Http;
   + using System.Web.Http.Dependencies;
   + using System.Web.Mvc;

     namespace FooNamespace
     {
         public static class WebApiConfig
         {
         public static void Register(HttpConfiguration config)
         {
   +         IServiceProvider provider = Global.ServiceProvider;
   +
   +         // Web API 2 resolver
   +         var resolver = new ServiceProviderDependencyResolver(provider);
   +         config.DependencyResolver = resolver;
   +
   +         // System.Web.Mvc resolver
   +         ServiceScopeModule.SetServiceProvider(provider);
   +         DependencyResolver.SetResolver(resolver);
   ```

8. Configure services as needed in `Startup.cs`.

Now, you should be able to register dependent classes and use them as
constructor parameters in your controllers, services, and repositories to
receive instances of those classes from the DI system. Read more about
Microsoft DI in the [docs][di-docs].

[di-docs]: https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection

## Consider using System.Text.Json for serialization

ASP.NET uses Newtonsoft.Json for JSON serialization, but modern .NET, and
therefore ASP.NET Core, now uses the built-in `System.Text.Json` library by
default. `System.Text.Json` is available as a package for .NET Framework, so
you can begin to use it in your .NET Framework projects before migration.

Some reasons you may want to switch to `System.Text.Json`:

- It has much lower memory usage and faster parsing and serialization times
- It is built-in to the .NET SDK, so security patches are included with the
  .NET runtime patches; once you migrate, you don't have to track a separate
  dependency for security updates.

Here are some considerations for whether you can switch.

### System.Text.Json is much stricter by default.

For example, configuration of converting from property names' `PascalCase` to
`camelCase` is explicit in `System.Text.Json`, but is implicit in `Newtonsoft.Json`. This can
cause errors if your server changes from being case-insensitive to
case-sensitive. ASP.NET Core configures deserialization to be case-insensitive
by default, but other areas in your application that also use JSON will need
to be configured either also to be case-insensitive or to ensure that the
formats between old and new clients and the old and new servers all match. This
includes reading JSON from databases, files, message queues or other storage.

### Polymorphic or dynamically-typed objects require custom annotations or converters.

`Newtonsoft.Json` has a [feature][nsj-poly], `TypeNameHandling`, that allows
you to put the type of the object to serialize within the payload. When
parsing, the server will use that field to determine the type of the object to
deserialize. Because this type discriminator is unrestricted and
user-controlled, this allows remote code execution. So for
[security reasons][stj-sec], the .NET team did not provide the equivalent in
`System.Text.Json` when it was introduced.

Now (since .NET 7), there is [support for polymorhphic types][stj-poly]
by declaring which subtypes are allowed to be deserialized from a particular
base class. So using polymorphic types, but it requires annotating your POCOs
with `System.Text.Json` attributes, or otherwise using a custom type converter

In short, if:
- your model types do not have any Newtonsoft-specific converters or attributes,
- you don't use `Dictionary` properties with keys that are implicitly case-sensitive, and
- you do not have any polymorphic types,

then you may be able to migrate to `System.Text.Json` without any problems. Start
by updating your clients to use `System.Text.Json` for serialization, and then
use the default settings for serialization when upgrading to ASP.NET Core to
take advantage of the `System.Text.Json` performance gains on the server.

If you cannot do this all at once, you may want to try selectively choosing the
serializer for each controller, as outlined in [this blog post][stj-select-controller].

[nsj-poly]: https://www.newtonsoft.com/json/help/html/SerializeTypeNameHandling.htm
[stj-sec]: https://github.com/dotnet/corefx/issues/41347#issuecomment-535779492
[stj-poly]: https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/polymorphism#polymorphic-type-discriminators
[stj-select-controller]: https://blogs.taiga.nl/martijn/posts/system-text-json-and-newtonsoft-json-side-by-side-in-asp-net-core/

## Add Tracing

You can use [distributed tracing][tracing-docs] to correlate requests and logs
across various services. This can be used with observability tools in several
ways. For example, it can help onboard new developers by letting them follow
the flow of individual requests through a whole system beyond debugger
breakpoint barriers, and operators can use it to identify problematic upstream
services. (I'll write more on exactly how I've done that at another time.)

Tracing comes in two parts: instrumentation and consumption. In .NET, you
interact with traces using the `Activity` class and other related classes.
Library and framework authors can instrument their code with activities, and
developers can consume the diagnostic data by configuring their applications
listen for certain activities from those libraries.

[tracing-docs]: https://learn.microsoft.com/en-us/dotnet/core/diagnostics/distributed-tracing

Modern .NET code has instrumentation built into many of the core libraries,
including `HttpClient`. (This is another reason why it's helpful for you to
switch your client applications to use HttpClient, since `HttpClient` are
instrumented, but `WebClient`, `WebRequest` et al. are not.) ASP.NET Core also
has instrumentation built-in _and_ is configured by default to automatically
propagate trace context passed via HTTP headers. Various monitoring tools, like
OpenTelemetry, Application Insights and Serilog, can consume and export to
other monitoring platforms.

### Why add tracing?

Distributed traces are most useful when all components in a request chain
propagate trace context. This can be frustrating, since distributed traces are
most useful when all applications participate; if a request goes from A -> B ->
C -> D, but B doesn't participate in distributed tracing, you get two traces, A
and C -> D, which is much less useful. As I mentioned, ASP.NET Core does this
automatically for HTTP requests, but ASP.NET does not.

While you are in the middle of migrating your services from ASP.NET to ASP.NET
Core, it may be useful to still have tracing available on your older services.
There are various ways to make ASP.NET propagate this context, but here is one
way, assuming that you are using W3C Trace Context to pass trace information.

### How to add tracing to your application

1. Add the `System.Diagnostics.DiagnosticSource` and `System.Reactive.Core` packages to your ASP.NET project.
2. Create a new file, `App_Start\DiagnosticsConfig.cs` with the following content:

   ```cs
   // App_Start/DiagnosticsConfig.cs
   using System;
   using System.Collections.Generic;
   using System.Diagnostics;
   using System.Net;
   using System.Reflection;
   namespace FooNamespace
   {
       public class DiagnosticsConfig
       {
           // Set up an ActivitySource
           private static readonly AssemblyName AssemblyName =
               typeof(DiagnosticsConfig).Assembly.GetName();

           internal static readonly ActivitySource ActivitySource =
               new ActivitySource(AssemblyName.Name, AssemblyName.Version.ToString());

           public static void StartDiagnosticsListeners()
           {
               // Set the format to W3C
               Activity.DefaultIdFormat = ActivityIdFormat.W3C;
               Activity.ForceDefaultIdFormat = true;
               // Start the ActivityListener
               ActivitySource.AddActivityListener(new ActivityListener()
               {
                   Sample = (ref ActivityCreationOptions<ActivityContext> _) =>
                       ActivitySamplingResult.AllData,
                       ShouldListenTo = _ => true,
               });

               // Using the Subscribe(IObservable<T>, Action<T>) extension method from
               // the System.Reactive.Core package.
               var subscription = DiagnosticListener.AllListeners.Subscribe(delegate (DiagnosticListener listener)
               {
                   if (listener.Name == "System.Net.Http.Desktop")
                   {
                       listener.Subscribe(delegate (KeyValuePair<string, object> diagnostic)
                       {
                           if (
                               diagnostic.Key == "System.Net.Http.Desktop.HttpRequestOut.Start"
                               && Activity.Current != null
                           )
                           {
                               // Set the parent to workaround HttpDiagnosticListener adding
                               // an extra intermediate span.
                               HttpWebRequest request = diagnostic.Value.GetType()
                                   .GetRuntimeProperty("Request")
                                   .GetValue(diagnostic.Value) as HttpWebRequest;

                               request.Headers.Add("traceparent", Activity.Current.ParentId);

                               var tracestate = Activity.Current.TraceStateString;
                               if (!string.IsNullOrWhiteSpace(tracestate))
                               {
                                   request.Headers.Add("tracestate", tracestate);
                               }
                           }
                       });
                   }
               });
           }
       }
   }
   ```

3. In `Global.asax.cs`, start the listeners you created in the previous step in the `Application_Start()` method, and add the
   `Application_BeginRequest()` and `Application_EndRequest()` methods from the snippet below:

   ```cs
   // Global.asax.cs
   using System.Diagnostics; // Add this line
   using System.Web;
   using System.Web.Http;
   using System.Web.Mvc;
   using System.Web.Optimization;
   using System.Web.Routing;
   namespace FooNamespace
   {
       public class Global : HttpApplication
       {
           protected void Application_Start()
           {
               // <Other startup boilerplate>
               // ...
               // Register your DiagnosticSource to send traceparent on outgoing requests.
               DiagnosticsConfig.StartDiagnosticsListeners(); // Add this line
           }

           // Add this method
           protected void Application_BeginRequest()
           {
               // Parse traceparent on incoming requests.
               var ctx = HttpContext.Current;
               const string operationName = "FooNamespace.AspNet.HandleRequest";
               var traceParent = ctx.Request.Headers["traceparent"];
               var traceState = ctx.Request.Headers["tracestate"];
               Activity activity = (traceParent != null && ActivityContext.TryParse(traceParent, traceState, out ActivityContext activityCtx))
                   ? DiagnosticsConfig.ActivitySource.StartActivity(operationName, ActivityKind.Server,
               activityCtx)
                   : DiagnosticsConfig.ActivitySource.StartActivity(operationName, ActivityKind.Server);
               ctx.Items["activity"] = activity;
           }

           // Add this method
           protected void Application_EndRequest()
           {
               // Clean up the Activity at the end of the request.
               var ctx = HttpContext.Current;
               var activity = ctx.Items["activity"] as Activity;
               activity?.Stop();
           }
       }
   }
   ```

After doing that, your ASP.NET should create activities on incoming HTTP
requests and HttpClient methods should propagate the trace IDs on outgoing HTTP
requests. Configuring your application to export this data is a separate step,
but this should make your application work with any monitoring tool that uses
Activity classes, like [OpenTelemetry exporters][otel-netfx].

[otel-netfx]: https://opentelemetry.io/docs/languages/dotnet/exporters/#non-aspnet-core

## Conclusion

This is mostly a reference that I've collected over time that I can look back
on when I've forgotten. I hope this is useful to you!

Do you have any tips on migrating .NET Framework web or console applications to
modern .NET? I'd like to hear them! Feel free to [reach out on
Mastodon](https://fosstodon.org/@iinuwa)!
