# Using Endpoints

Steeltoe provides a base set of endpoint functionality, along with several implementations for exposing the endpoints over HTTP. HTTP implementations are provided with ASP.NET Core middleware. To expose the core endpoint functionality over some protocol other than HTTPS, you can provide your own implementation.

## Reference Materials

In this section, it is helpful to understand the following:

* How the .NET [Configuration service](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration) works and an understanding of the `ConfigurationBuilder` and how to add providers to the builder to configure the endpoints.
* How the ASP.NET Core [`Startup`](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/startup) class is used in configuring the application services for the app. Pay particular attention to the usage of the `ConfigureServices()` and `Configure()` methods.

## Endpoint Listing

The following table describes the available Steeltoe management endpoints that can be used in an application:

| ID|Description |
| --- | --- |
| `hypermedia` | Provides the hypermedia endpoint for discovery of all available endpoints. |
| `cloudfoundry` | Enables the management endpoint integration with Cloud Foundry. |
| `health` | Customizable endpoint that gathers application health information. |
| `info` | Customizable endpoint that gathers arbitrary application information (such as Git Build info). |
| `loggers` | Gathers existing loggers and allows modification of logging levels. |
| `trace` | Gathers a configurable set of trace information (such as the last 100 HTTP requests). |
| `refresh` | Triggers the application configuration to be reloaded. |
| `env` | Reports the keys and values from the application's configuration. |
| `mappings` | Reports the configured ASP.NET routes and route templates. |
| `metrics` | Reports the collected metrics for the application. |
| `dump` | Generates and reports a snapshot of the application's threads (Windows only). |
| `heapdump` | Generates and downloads a mini-dump of the application. |

Each endpoint has an associated ID. When you want to expose that endpoint over HTTP, that ID is used in the mapped URL that exposes the endpoint. For example, the `health` endpoint is mapped to `/health`.

## Add NuGet References

To use the management endpoints, you need to add a reference to the appropriate Steeltoe NuGet, based on the type of the application you are building and what dependency injector you have, if any.

The following table describes the available packages:

| Package | Description | .NET Target |
| --- | --- | --- |
| `Steeltoe.Management.Abstractions` | Interfaces and objects used for extensibility. | .NET Standard 2.0 |
| `Steeltoe.Management.EndpointBase` | Base functionality, no dependency injection, and no HTTP middleware. | .NET Standard 2.0 |
| `Steeltoe.Management.EndpointCore` | Includes `EndpointBase`, adds ASP.NET Core DI, and includes HTTP middleware. | ASP.NET Core 3.1+ |
| `Steeltoe.Management.CloudFoundryCore` | Includes `EndpointCore` and Cloud Foundry related functionality. | ASP.NET Core 3.1+ |
| `Steeltoe.Management.KubernetesCore` | Includes `EndpointCore` and Kubernetes related functionality. | ASP.NET Core 3.1+ |

To add this type of NuGet to your project, add a `PackageReference` resembling the following:

```xml
<ItemGroup>
...
    <PackageReference Include="Steeltoe.Management.EndpointCore" Version= "3.0.0"/>
...
</ItemGroup>
```

## Configure Global Settings

Endpoints can be configured by using the normal .NET [configuration service](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration). You can globally configure settings that apply to all endpoints as well as configure settings that are specific to a particular endpoint.

All management endpoint settings should be placed under the prefix with the `Management:Endpoints` key. Any settings found under this prefix apply to all endpoints globally.

Settings that you want to apply to specific endpoints should be placed under the prefix with the `Management:Endpoints:` key plus the ID of the endpoint (for example, `Management:Endpoints:Health`). Any settings you apply to a specific endpoint override any settings applied globally.

The following table describes the settings that you can apply globally:

| Key | Description | Default |
| --- | --- | --- |
| `Enabled` | Whether to enable all management endpoints. | `true` |
| `Path` | The path prefix applied to all endpoints when exposed over HTTP. | `/` |

## Exposing Endpoints

Since endpoints may contain sensitive information, only health and info are exposed by default. To change which endpoints are exposed, use the `include` and `exclude` properties:

| Property | Default |
| --- | --- |
| `Exposure:Include` | [`info`, `health`] |
| `Exposure:Exclude` | |

>NOTE: Each setting above must be prefixed with `Management:Endpoints:actuator`. To select all endpoints,
`*`  can be used. For example, to expose everything except `env` and `refresh`, use the following property:

```json
"Management": {
    "Endpoints": {
        "Actuator":{
            "Exposure": {
                "Include": [ "*" ],
                "Exclude": [ "env", "refresh"]
            }
        }
    }
}
```

The sections that follow show the settings that you can apply to specific endpoints.

## HTTP Access

To expose any of the management endpoints over HTTP in an ASP.NET Core application:

1. Add a reference to `Steeltoe.Management.EndpointCore` or `Steeltoe.Management.CloudFoundryCore`.
1. Configure endpoint settings, as needed (for example, `appsettings.json`).
1. Add any additional "contributors" to the service container (for example, `AddSingleton<IHealthContributor, CustomHealthContributor>()`).
1. `Add` the actuator endpoint to the service container (for example, `AddHealthActuator()`).
1. `Use` the actuator middleware to provide HTTP access (for example, `UseInfoActuator()`).

>NOTE: Each endpoint uses the same host and port as the application. The default path to each endpoint is specified in its section on this page, along with specific `Add` and `Use` method names.

Extensions for both `IHostBuilder` and `IWebHostBuilder` are included to configure actuators with a single line of code in `program.cs`:

```csharp
    public static void Main(string[] args)
    {
        BuildWebHost(args).Run();
    }
    public static IHost BuildHost(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .AddAllActuators()
            .Build();
```

>NOTE: `AddAllActuators()` and `AddLoggingActuator()` automatically configure the `DynamicConsoleLogger`. To use the dynamic Serilog console logger, be sure to do so before adding actuators.

If you prefer to configure the actuators in `Startup.cs`, extensions are provided for `IServiceCollection` and `IApplicationBuilder` to configure and activate the actuator middleware implementations. To use all of the Steeltoe endpoints, use `AddAllActuators()` and `MapAllActuators` to add them all at once instead of including each individually, as follows:

```csharp
public class Startup
{
    public IConfiguration Configuration { get; }
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }
    public void ConfigureServices(IServiceCollection services)
    {
        ...
        // Add management endpoint services like this
        services.AddAllActuators(Configuration);
        ...
    }
    public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
        ...
        app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllerRoute(
                    name: "default",
                    pattern: "{controller=Home}/{action=Index}/{id?}");

                // Add management endpoints into pipeline like this
                endpoints.MapAllActuators();

                // ... Other mappings
            });
        ...
    }
}
```

>NOTE: The order in which you add middleware to the [ASP.NET Core pipeline](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/) is important. We recommend that you add the Steeltoe management endpoints before others to ensure proper operation.
