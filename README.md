# NotS-WAPP-Headers
# Contents
- [NotS-WAPP-Headers](#nots-wapp-headers)
- [Contents](#contents)
- [About](#about)
- [Reference](#reference)
- [Setting response headers in .NET](#setting-response-headers-in-net)
  - [Web.config](#webconfig)
  - [The Response object](#the-response-object)
- [Built-in response header middleware](#built-in-response-header-middleware)

# About

This repository contains a simple demonstration of the usage of response headers in a .NET Core web API. Furthermore, this README gives more context around response headers and the raw values they can contain.

# Reference

Most information on this page originates from [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers). To see more information about .NET's built in middleware that facilitates most important response headers, have a look at [Microsoft's API documentation](https://docs.microsoft.com/en-us/dotnet/api/?view=net-5.0).

# Setting response headers in .NET

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

# Built-in response header middleware