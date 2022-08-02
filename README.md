# Output Caching for ASP.NET Core 6.0

A copy of .NET 7.0 Output Caching middleware, targeting .NET 6.0.

## Warning !!!

This package is not supported and might be removed in the future. It's goal is to provide a way to test the Output Caching features that will ship in ASP.NET Core 7.0 but on .NET 6.0. Any improvement made to the official version will be ported here.

## Sample usage - Minimal APIs

#### Program.cs

```c#
using System.Globalization;
using Microsoft.AspNetCore.OutputCaching;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOutputCache(options =>
{
    // Define policies for all requests which are not configured per endpoint or per request
    options.AddBasePolicy(builder => builder.With(c => c.HttpContext.Request.Path.StartsWithSegments("/js")).Expire(TimeSpan.FromDays(1)));
    options.AddBasePolicy(builder => builder.With(c => c.HttpContext.Request.Path.StartsWithSegments("/js")).NoCache());

    options.AddPolicy("NoCache", b => b.NoCache());
});

var app = builder.Build();

app.UseOutputCache();

app.MapGet("/", Gravatar.WriteGravatar);

app.MapGet("/cached", Gravatar.WriteGravatar).CacheOutput();

app.MapGet("/nocache", Gravatar.WriteGravatar).CacheOutput(x => x.NoCache());

app.MapGet("/profile", Gravatar.WriteGravatar).CacheOutput("NoCache");

app.MapGet("/attribute", [OutputCache(PolicyName = "NoCache")] () => Gravatar.WriteGravatar);

// Only available in dotnet 7
//var blog = app.MapGroup("blog").CacheOutput(x => x.Tag("blog"));
//blog.MapGet("/", Gravatar.WriteGravatar);
//blog.MapGet("/post/{id}", Gravatar.WriteGravatar).CacheOutput(x => x.Tag("blog", "byid")); // Calling CacheOutput() here overwrites the group's policy

app.MapPost("/purge/{tag}", async (IOutputCacheStore cache, string tag) =>
{
    // POST such that the endpoint is not cached itself

    await cache.EvictByTagAsync(tag, default);
});

// Cached entries will vary by culture, but any other additional query is ignored and returns the same cached content
app.MapGet("/query", Gravatar.WriteGravatar).CacheOutput(p => p.VaryByQuery("culture"));

app.MapGet("/vary", Gravatar.WriteGravatar).CacheOutput(c => c.VaryByValue((context) => new KeyValuePair<string, string>("time", (DateTime.Now.Second % 2).ToString(CultureInfo.InvariantCulture))));

long requests = 0;

// Locking is enabled by default
app.MapGet("/lock", async (context) =>
{
    await Task.Delay(1000);
    await context.Response.WriteAsync($"<pre>{requests++}</pre>");
}).CacheOutput(p => p.AllowLocking(false).Expire(TimeSpan.FromMilliseconds(1)));

// Etag
app.MapGet("/etag", async (context) =>
{
    // If the client sends an If-None-Match header with the etag value, the server
    // returns 304 if the cache entry is fresh instead of the full response

    var etag = $"\"{Guid.NewGuid():n}\"";
    context.Response.Headers.ETag = etag;

    await Gravatar.WriteGravatar(context);

    var cacheContext = context.Features.Get<IOutputCacheFeature>()?.Context;

}).CacheOutput();

// When the request header If-Modified-Since is provided, return 304 if the cached entry is older
app.MapGet("/ims", Gravatar.WriteGravatar).CacheOutput();

await app.RunAsync();

```

## Sample usage - ASP.NET Core MVC

Enabling output cache for an MVC action:

#### Program.cs

```c#
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllersWithViews();
builder.Services.AddOutputCache();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
}
app.UseStaticFiles();

app.UseRouting();

app.UseOutputCache();

app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

#### HomeController.cs

```c#
public class HomeController : Controller
{
    [OutputCache(Duration = 5)]
    public IActionResult Index()
    {
        return View();
    }
}
```

## Sample usage - Razor Page Model

Enabling output cache for a Razor Page:

#### Program.cs

```c#
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddRazorPages();
builder.Services.AddOutputCache();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
}
app.UseStaticFiles();

app.UseRouting();

app.UseOutputCache();

app.UseAuthorization();

app.MapRazorPages();

app.Run();
```

#### Index.cshtml.cs

```c#
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.AspNetCore.OutputCaching;

namespace WebApplication4.Pages
{
    [OutputCache(Duration = 5)]
    public class IndexModel : PageModel
    {
        public void OnGet()
        {

        }
    }
}
```
