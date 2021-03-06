# NotS-WAPP-Headers
# Contents
- [NotS-WAPP-Headers](#nots-wapp-headers)
- [Contents](#contents)
- [About](#about)
- [Reference](#reference)
- [Setting response headers in .NET](#setting-response-headers-in-net)
  - [Web.config](#webconfig)
  - [The Response object](#the-response-object)
- [Common response headers](#common-response-headers)
  - [Caching](#caching)
  - [Cookies](#cookies)
  - [Cors](#cors)
  - [Security](#security)
    - [HSTS](#hsts)
    - [Unset Server header](#unset-server-header)

# About

This repository contains a simple demonstration of the usage of response headers in a .NET Core web API. Furthermore, this README gives more context around response headers and the raw values they can contain.

# Reference

Most information on this page originates from [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers). To see more information about .NET's built in middleware that facilitates most important response headers, have a look at [Microsoft's API documentation](https://docs.microsoft.com/en-us/dotnet/api/?view=net-5.0).

# Setting response headers in .NET

You can manually set HTTP response headers in .NET Core. Be mindful though that there are a lot of built-in middleware functions in this framework that are very likely to set the headers you desire. Some of these middleware functions are touched upon [here]((#common-response-headers)). The Microsoft documentation about .NET Core middleware can be found [here](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-5.0).
## Web.config
> **!** This only works with IIS Express

You can create a [`Web.config`](Web.config) file in your root directory and add response headers like this:

```xml
<?xml version="1.0" encoding="utf-8"?>

<configuration>
    <system.web>
        <compilation debug="true"  />
        <httpRuntime  />
    </system.web>
    <system.webServer>
        <httpProtocol>
            <customHeaders>
                <add name="Test" value="Value" />
            </customHeaders>
        </httpProtocol>
    </system.webServer>
</configuration>
```

The downside of this is that it's a predefined file and will often not work out of the box during deployment.

## The Response object

You can use the `Response.Headers.Add(..)` ([docs](https://docs.microsoft.com/en-us/dotnet/api/system.net.http.headers.httpheaders.add?view=net-5.0)) method to add response headers. You can either create your own middleware function in [`Startup.cs`](Startup.cs) that does this:

```cs
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("Test-2", "Value");
    await next();
});
```

Or you can set the response header in [a controller method](Controllers/WeatherForecastController.cs), which by default has access to the `Response` object because it inherits from `ControllerBase`:

```cs
[HttpGet]
public IEnumerable<WeatherForecast> Get()
{
    Response.Headers.Add("Test-3", "Value");

    var rng = new Random();
    return Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            Date = DateTime.Now.AddDays(index),
            TemperatureC = rng.Next(-20, 55),
            Summary = Summaries[rng.Next(Summaries.Length)]
        })
        .ToArray();
}
```

If you're using a header from the HTTP spec, you might be able to set a typed header instead:

```cs
app.Use(async (context, next) =>
{
    context.Response.GetTypedHeaders().CacheControl = new CacheControlHeaderValue()
    {
        Public = true,
        MaxAge = TimeSpan.FromDays(30)
    };
    await next();
});
```

The `Microsoft.Net.Http.Headers` namespace ([docs](https://docs.microsoft.com/en-us/dotnet/api/microsoft.net.http.headers?view=aspnetcore-5.0)) contains typed classes for headers that support multiple parameters.

# Common response headers

This section is separated in a few categories. These categories correlate with the categories that the [MDN documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers) uses. All examples are also added to the [`Startup.cs`](Startup.cs) file.

## Caching

You can use the `Microsoft.Net.Http.Headers.CacheControlHeaderValue` typed header to set this header:

```cs
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.Use(async (context, next) =>
    {
        context.Response.GetTypedHeaders().CacheControl = new CacheControlHeaderValue()
        {
            Public = true,
            MaxAge = TimeSpan.FromDays(30)
        };
        await next();
    });
}
```

Of course, you can also set it manually:

```cs
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.Use(async (context, next) =>
    {
        context.Response.Headers.Add("Cache-Control", "public, max-age 2592000");
    });
}
```

The resulting `Cache-Control` header looks like this:

```
Cache-Control:	public, max-age=2592000
```

> You can also enable server-side caching with `app.useResponseCaching()`. See [Microsoft's response caching middleware documentation](https://docs.microsoft.com/en-us/aspnet/core/performance/caching/middleware?view=aspnetcore-5.0) for more details.

## Cookies

You can configure the global cookie policy with `services.Configure<CookiePolicyOptions>` ([docs](https://docs.microsoft.com/en-us/aspnet/core/security/gdpr?view=aspnetcore-5.0)).

```cs
public void ConfigureServices(IServiceCollection services)
{
    // Cookie policy
    services.Configure<CookiePolicyOptions>(options =>
    {
        options.MinimumSameSitePolicy = SameSiteMode.None;
        options.Secure = CookieSecurePolicy.Always;
        options.HttpOnly = HttpOnlyPolicy.Always;
    });
}
```

Adding `app.UseCookiePolicy()` will enforce the policy whenever a cookie is added to the `Response` object:

```cs
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // Cookie policy
    app.UseCookiePolicy();
}
```

You can add a cookie to a `Response` object like this ([docs](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.iresponsecookies.append?view=aspnetcore-5.0)):

```cs
[HttpGet]
public IEnumerable<WeatherForecast> Get()
{
    /** Add a cookie to the response
     * This will add the Set-Cookie HTTP header and add the cookie policy defined in Startup.cs. The policy can
     * be overridden by supplying a Microsoft.AspNetCore.Http.CookieOptions object to the function below.
     */
    Response.Cookies.Append("cookieKey", "cookieValue");

    var rng = new Random();
    return Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            Date = DateTime.Now.AddDays(index),
            TemperatureC = rng.Next(-20, 55),
            Summary = Summaries[rng.Next(Summaries.Length)]
        })
        .ToArray();
}
```

The resulting `Set-Cookie` header looks like this:

```
Set-Cookie:	cookieKey=cookieValue; path=/; secure; samesite=none; httponly
```

As you can see, the defined cookie policy was added to the header.

> Also noteworthy: using `app.UseAuthentication()` or `app.UseSession()` will likely add a `Set-Cookie` header to the response as well. If this is the case, you should call `app.UseCookiePolicy()` before calling these methods. This will ensure the cookie policy is also applied to these cookies.

## Cors

Configuring cors is pretty straight-forward ([docs](https://docs.microsoft.com/en-us/aspnet/core/security/cors?view=aspnetcore-5.0)). You can use the [cors policy builder](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.cors.infrastructure.corspolicybuilder?view=aspnetcore-5.0):

```cs
public void ConfigureServices(IServiceCollection services)
{
    // Cors
    services.AddCors(options =>
    {
        options.AddDefaultPolicy(builder =>
        {
            builder.WithOrigins("https://localhost:5000")
                .AllowAnyMethod()
                .AllowAnyHeader()
                .AllowCredentials();
        });
    });
}
```

Enable cors (on all endpoints with the default policy) like this:

```cs
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // Cors
    app.UseCors();
}
```

It's also possible to create a named policy if you want to use different cors policies for different endpoints. The builder will look like this:

```cs
options.AddPolicy("policyName", builder =>
{
    builder.WithOrigins("https://localhost:5000")
        .AllowAnyMethod()
        .AllowAnyHeader()
        .AllowCredentials();
});
```

You can then add the policy to specific endpoints with the `[EnableCors]` attribute like this:

```cs
[HttpGet]
[EnableCors("policyName")]
public IEnumerable<WeatherForecast> Get()
{
    ...
}
```

For the default policy, you can just use `[EnableCors]` without parameters.

> Make sure the `app.UseCors();` statement is in between `app.UseRouting();` and `app.UseEndpoints(...);` if you want to use endpoint specific cors policies.

If your `Origin` matches the allowed origins in the cors policy, the response headers will look like this:

```
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: https://localhost:5000
```

## Security

### HSTS

If you want to instruct the client to request over HTTPS only you can enable HSTS. This will add the `Strict-Transport-Security` response header. By default, this doesn't work on local addresses because it could cause issues. To enable HSTS, add the following configuration:

```cs
public void ConfigureServices(IServiceCollection services)
{
    // HSTS / Scrict-Transport-Security
    services.AddHsts(options =>
    {
        options.Preload = true;
        options.IncludeSubDomains = true;
        options.MaxAge = TimeSpan.FromDays(30);
    });
}
```

Next, enable it for all responses:

```cs
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // HSTS
    app.UseHsts();
}
```

This will add the following header to all responses (unless the server runs on local addresses):

```
Strict-Transport-Security: max-age=2592000; includeSubDomains; preload
```

### Unset Server header

.NET Core adds the `Server: Kestrel` header by default. To prevent targeted attacks, you can unset this in your [`Program.cs`](Program.cs):

```cs
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
            
            // Add this to remove the server header
            webBuilder.UseKestrel(options =>
            {
                options.AddServerHeader = false;
            });
        });
```

