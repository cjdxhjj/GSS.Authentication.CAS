# GSS.Authentication.CAS

CAS Authentication Middleware for OWIN & ASP.NET Core

[![Build status](https://ci.appveyor.com/api/projects/status/uk7kwjvo1e6yl33m?svg=true)](https://ci.appveyor.com/project/akunzai/gss-authentication-cas)

## NuGet Packages

- [GSS.Authentication.CAS.Core ![NuGet version](https://img.shields.io/nuget/v/GSS.Authentication.CAS.Core.svg?style=flat-square)](https://www.nuget.org/packages/GSS.Authentication.CAS.Core/)
- [GSS.Authentication.CAS.Owin ![NuGet version](https://img.shields.io/nuget/v/GSS.Authentication.CAS.Owin.svg?style=flat-square)](https://www.nuget.org/packages/GSS.Authentication.CAS.Owin/)
- [GSS.Authentication.CAS.AspNetCore ![NuGet version](https://img.shields.io/nuget/v/GSS.Authentication.CAS.AspNetCore.svg?style=flat-square)](https://www.nuget.org/packages/GSS.Authentication.CAS.AspNetCore/)
- [GSS.Authentication.CAS.DistributedCache ![NuGet version](https://img.shields.io/nuget/v/GSS.Authentication.CAS.DistributedCache.svg?style=flat-square)](https://www.nuget.org/packages/GSS.Authentication.CAS.DistributedCache/)
- [GSS.Authentication.CAS.RuntimeCache ![NuGet version](https://img.shields.io/nuget/v/GSS.Authentication.CAS.RuntimeCache.svg?style=flat-square)](https://www.nuget.org/packages/GSS.Authentication.CAS.RuntimeCache/)

## Installation

OWIN

```shell
# Package Manager
Install-Package GSS.Authentication.CAS.Owin
# .NET CLI
dotnet add package GSS.Authentication.CAS.Owin
```

ASP.NET Core 1.x

```shell
# Package Manager
Install-Package GSS.Authentication.CAS.AspNetCore -Version 1.2.0
# .NET CLI
dotnet add package GSS.Authentication.CAS.AspNetCore --version 1.2.0
```

ASP.NET Core 2.x

```shell
# Package Manager
Install-Package GSS.Authentication.CAS.AspNetCore
# .NET CLI
dotnet add package GSS.Authentication.CAS.AspNetCore
```

## Usage

### Single-Sign-On

OWIN

```csharp
public class Startup
{
    public void Configuration(IAppBuilder app)
    {
        var env = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Production";
        var configuration = new ConfigurationBuilder()
            .AddJsonFile($"appsettings.json", reloadOnChange: true)
            .AddJsonFile($"appsettings.{env}.json", optional: true, reloadOnChange: true);
            .Build();
        app.UseCasAuthentication(options =>
        {
            options.CasServerUrlBase = configuration["Authentication:CAS:ServerUrlBase"];
            var protocolVersion = configuration.GetValue("Authentication:CAS:ProtocolVersion", 3);
            if (protocolVersion != 3)
            {
                switch (protocolVersion)
                {
                    case 1:
                        options.ServiceTicketValidator = new Cas10ServiceTicketValidator(options);
                        break;
                    case 2:
                        options.ServiceTicketValidator = new Cas20ServiceTicketValidator(options);
                        break;
                }
            }
            options.Provider = new CasAuthenticationProvider
            {
                OnCreatingTicket = context =>
                {
                    // add claims from CasIdentity.Assertion ?
                    var assertion = (context.Identity as CasIdentity)?.Assertion;
                    if (assertion == null) return Task.CompletedTask;
                    context.Identity.AddClaim(new Claim(context.Identity.NameClaimType, assertion.PrincipalName));
                    if (assertion.Attributes.TryGetValue("email", out var email))
                    {
                        context.Identity.AddClaim(new Claim(ClaimTypes.Email, email));
                    }
                    if (assertion.Attributes.TryGetValue("display_name", out var displayName))
                    {
                        context.Identity.AddClaim(new Claim(ClaimTypes.GivenName, displayName));
                    }
                    return Task.CompletedTask;
                }
            };
        });
    }
}
```

ASP.NET Core 1.x

```csharp
public class Startup
{
    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.Configure<CasAuthenticationOptions>(options =>
        {
            options.CasServerUrlBase = Configuration["Authentication:CAS:ServerUrlBase"];
            var protocolVersion = Configuration.GetValue("Authentication:CAS:ProtocolVersion", 3);
            if (protocolVersion != 3)
            {
                switch (protocolVersion)
                {
                    case 1:
                        options.ServiceTicketValidator = new Cas10ServiceTicketValidator(options);
                        break;
                    case 2:
                        options.ServiceTicketValidator = new Cas20ServiceTicketValidator(options);
                        break;
                }
            }
            options.Events = new CasEvents
            {
                OnCreatingTicket = context =>
                {
                    // add claims from CasIdentity.Assertion ?
                    var assertion = (context.Principal as ICasPrincipal)?.Assertion;
                    if (assertion == null) return Task.CompletedTask;
                    var identity = context.Principal.Identity as ClaimsIdentity;
                    if (identity == null) return Task.CompletedTask;
                    identity.AddClaim(new Claim(identity.NameClaimType, assertion.PrincipalName));
                    if (assertion.Attributes.TryGetValue("email", out var email))
                    {
                        identity.AddClaim(new Claim(ClaimTypes.Email, email));
                    }
                    if (assertion.Attributes.TryGetValue("display_name", out var displayName))
                    {
                        identity.AddClaim(new Claim(ClaimTypes.GivenName, displayName));
                    }
                    return Task.CompletedTask;
                }
            };
        });
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseCasAuthentication();
    }
}
```

ASP.NET Core 2.x

```csharp
public class Startup
{
    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
        .AddCAS(options =>{
            options.CallbackPath = "/signin-cas";
            options.CasServerUrlBase = Configuration["Authentication:CAS:ServerUrlBase"];
            var protocolVersion = Configuration.GetValue("Authentication:CAS:ProtocolVersion", 3);
            if (protocolVersion != 3)
            {
                switch (protocolVersion)
                {
                    case 1:
                        options.ServiceTicketValidator = new Cas10ServiceTicketValidator(options);
                        break;
                    case 2:
                        options.ServiceTicketValidator = new Cas20ServiceTicketValidator(options);
                        break;
                }
            }
            options.Events = new CasEvents
            {
                OnCreatingTicket = context =>
                {
                    // add claims from CasIdentity.Assertion ?
                    var assertion = context.Assertion;
                    if (assertion == null) return Task.CompletedTask;
                    if (!(context.Principal.Identity is ClaimsIdentity identity)) return Task.CompletedTask;
                    identity.AddClaim(new Claim(identity.NameClaimType, assertion.PrincipalName));
                    if (assertion.Attributes.TryGetValue("email", out var email))
                    {
                        identity.AddClaim(new Claim(ClaimTypes.Email, email));
                    }
                    if (assertion.Attributes.TryGetValue("display_name", out var displayName))
                    {
                        identity.AddClaim(new Claim(ClaimTypes.GivenName, displayName));
                    }
                    return Task.CompletedTask;
                }
            };
        });
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseAuthentication();
    }
}
```

### Single-Sign-Out

OWIN

```csharp
public class Startup
{
    public void Configuration(IAppBuilder app)
    {
        var env = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Production";
        var configuration = new ConfigurationBuilder()
            .AddJsonFile($"appsettings.json", reloadOnChange: true)
            .AddJsonFile($"appsettings.{env}.json", optional: true, reloadOnChange: true);
            .Build();
        var sessionStore = new AuthenticationSessionStoreWrapper(new RuntimeCacheServiceTicketStore());
        app.UseCasSingleSignOut(sessionStore);
        app.SetDefaultSignInAsAuthenticationType(CookieAuthenticationDefaults.AuthenticationType);
        app.UseCookieAuthentication(new CookieAuthenticationOptions
        {
            LoginPath = new PathString("/login"),
            LogoutPath = new PathString("/logout"),
            SessionStore = sessionStore,
            Provider = new CookieAuthenticationProvider
            {
                OnResponseSignOut = context =>
                {
                    // Single Sign-Out
                    var casUrl = new Uri(configuration["Authentication:CAS:ServerUrlBase"]);
                    var serviceUrl = context.Request.Uri.GetComponents(UriComponents.SchemeAndServer, UriFormat.Unescaped);
                    var redirectUri = new UriBuilder(casUrl);
                    redirectUri.Path += "/logout";
                    redirectUri.Query = $"service={Uri.EscapeDataString(serviceUrl)}";
                    var logoutRedirectContext = new CookieApplyRedirectContext(
                        context.OwinContext,
                        context.Options,
                        redirectUri.Uri.AbsoluteUri
                        );
                    context.Options.Provider.ApplyRedirect(logoutRedirectContext);
                }
            }
        });
        app.UseCasAuthentication(new CasAuthenticationOptions
        {
            CasServerUrlBase = configuration["Authentication:CAS:ServerUrlBase"],
            UseAuthenticationSessionStore = true,
            var protocolVersion = configuration.GetValue("Authentication:CAS:ProtocolVersion", 3);
            if (protocolVersion != 3)
            {
                switch (protocolVersion)
                {
                    case 1:
                        options.ServiceTicketValidator = new Cas10ServiceTicketValidator(options);
                        break;
                    case 2:
                        options.ServiceTicketValidator = new Cas20ServiceTicketValidator(options);
                        break;
                }
            }
            Provider = new CasAuthenticationProvider
            {
                OnCreatingTicket = context =>
                {
                    // add claims from CasIdentity.Assertion ?
                    var assertion = (context.Identity as CasIdentity)?.Assertion;
                    if (assertion == null) return Task.CompletedTask;
                    context.Identity.AddClaim(new Claim(context.Identity.NameClaimType, assertion.PrincipalName));
                    if (assertion.Attributes.TryGetValue("email", out var email))
                    {
                        context.Identity.AddClaim(new Claim(ClaimTypes.Email, email));
                    }
                    var displayName = assertion.Attributes["display_name"];
                    if (!StringValues.IsNullOrEmpty(displayName))
                    {
                        context.Identity.AddClaim(new Claim(ClaimTypes.GivenName, displayName));
                    }
                    return Task.CompletedTask;
                }
            }
        });
    }
}
```

ASP.NET Core 1.x

```csharp
public class Startup
{
    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        var serviceTicketStore = new DistributedCacheServiceTicketStore();
        var ticketStore = new TicketStoreWrapper(serviceTicketStore);

        services.AddSingleton<IServiceTicketStore>(serviceTicketStore);
        services.AddSingleton<ITicketStore>(ticketStore);
        services.AddAuthentication(options => options.SignInScheme = CookieAuthenticationDefaults.AuthenticationScheme);
        services.Configure<CookieAuthenticationOptions>(options =>
        {
            options.AutomaticAuthenticate = true;
            options.AutomaticChallenge = true;
            options.LoginPath = new PathString("/login");
            options.LogoutPath = new PathString("/logout");
            options.SessionStore = ticketStore;
            options.Events = new CookieAuthenticationEvents
            {
                OnSigningOut = context =>
                {
                    // Single Sign-Out
                    var casUrl = new Uri(Configuration["Authentication:CAS:ServerUrlBase"]);
                    var serviceUrl = new Uri(context.Request.GetEncodedUrl())
                        .GetComponents(UriComponents.SchemeAndServer, UriFormat.Unescaped);
                    var redirectUri = UriHelper.BuildAbsolute(
                        casUrl.Scheme,
                        new HostString(casUrl.Host, casUrl.Port),
                        casUrl.LocalPath, "/logout",
                        QueryString.Create("service", serviceUrl));
                    var logoutRedirectContext = new CookieRedirectContext(
                        context.HttpContext,
                        context.Options,
                        redirectUri,
                        context.Properties
                        );
                    context.Response.StatusCode = 204; //Prevent RedirectToReturnUrl
                    context.Options.Events.RedirectToLogout(logoutRedirectContext);
                    return Task.CompletedTask;
                }
            };
        });
        services.Configure<CasAuthenticationOptions>(options =>
        {
            options.CasServerUrlBase = Configuration["Authentication:CAS:ServerUrlBase"];
            var protocolVersion = Configuration.GetValue("Authentication:CAS:ProtocolVersion", 3);
            if (protocolVersion != 3)
            {
                switch (protocolVersion)
                {
                    case 1:
                        options.ServiceTicketValidator = new Cas10ServiceTicketValidator(options);
                        break;
                    case 2:
                        options.ServiceTicketValidator = new Cas20ServiceTicketValidator(options);
                        break;
                }
            }
            options.Events = new CasEvents
            {
                OnCreatingTicket = context =>
                {
                    // add claims from CasIdentity.Assertion ?
                    var assertion = (context.Principal as ICasPrincipal)?.Assertion;
                    if (assertion == null) return Task.CompletedTask;
                    var identity = context.Principal.Identity as ClaimsIdentity;
                    if (identity == null) return Task.CompletedTask;
                    identity.AddClaim(new Claim(identity.NameClaimType, assertion.PrincipalName));
                    if (assertion.Attributes.TryGetValue("email", out var email))
                    {
                        identity.AddClaim(new Claim(ClaimTypes.Email, email));
                    }
                    if (assertion.Attributes.TryGetValue("display_name", out var displayName))
                    {
                        identity.AddClaim(new Claim(ClaimTypes.GivenName, displayName));
                    }
                    return Task.CompletedTask;
                }
            };
        });
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseCasSingleSignOut();
        app.UseCookieAuthentication();
        app.UseCasAuthentication();
    }
}
```

ASP.NET Core 2.x

```csharp
public class Startup
{
    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        var serviceTicketStore = new DistributedCacheServiceTicketStore();
        var ticketStore = new TicketStoreWrapper(serviceTicketStore);

        services.AddSingleton<IServiceTicketStore>(serviceTicketStore);
        services.AddSingleton<ITicketStore>(ticketStore);
        services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
        .AddCookie(options =>
        {
            options.LoginPath = "/login";
            options.LogoutPath = "/logout";
            options.SessionStore = ticketStore;
            options.Events = new CookieAuthenticationEvents
            {
                OnSigningOut = context =>
                {
                    // Single Sign-Out
                    var casUrl = new Uri(Configuration["Authentication:CAS:ServerUrlBase"]);
                    var serviceUrl = new Uri(context.Request.GetEncodedUrl())
                        .GetComponents(UriComponents.SchemeAndServer, UriFormat.Unescaped);
                    var redirectUri = UriHelper.BuildAbsolute(
                        casUrl.Scheme,
                        new HostString(casUrl.Host, casUrl.Port),
                        casUrl.LocalPath, "/logout",
                        QueryString.Create("service", serviceUrl));
                    var logoutRedirectContext = new RedirectContext<CookieAuthenticationOptions>(
                        context.HttpContext,
                        context.Scheme,
                        context.Options,
                        context.Properties,
                        redirectUri
                    );
                    context.Response.StatusCode = 204; //Prevent RedirectToReturnUrl
                    context.Options.Events.RedirectToLogout(logoutRedirectContext);
                    return Task.CompletedTask;
                }
            };
        })
        .AddCAS(options =>{
            options.CallbackPath = "/signin-cas";
            options.CasServerUrlBase = Configuration["Authentication:CAS:ServerUrlBase"];
            var protocolVersion = Configuration.GetValue("Authentication:CAS:ProtocolVersion", 3);
            if (protocolVersion != 3)
            {
                switch (protocolVersion)
                {
                    case 1:
                        options.ServiceTicketValidator = new Cas10ServiceTicketValidator(options);
                        break;
                    case 2:
                        options.ServiceTicketValidator = new Cas20ServiceTicketValidator(options);
                        break;
                }
            }
            options.Events = new CasEvents
            {
                OnCreatingTicket = context =>
                {
                    // add claims from CasIdentity.Assertion ?
                    var assertion = context.Assertion;
                    if (assertion == null) return Task.CompletedTask;
                    if (!(context.Principal.Identity is ClaimsIdentity identity)) return Task.CompletedTask;
                    identity.AddClaim(new Claim(identity.NameClaimType, assertion.PrincipalName));
                    if (assertion.Attributes.TryGetValue("email", out var email))
                    {
                        identity.AddClaim(new Claim(ClaimTypes.Email, email));
                    }
                    if (assertion.Attributes.TryGetValue("display_name", out var displayName))
                    {
                        identity.AddClaim(new Claim(ClaimTypes.GivenName, displayName));
                    }
                    return Task.CompletedTask;
                }
            };
        });
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseCasSingleSignOut();
        app.UseAuthentication();
    }
}
```

## [Release Notes](https://github.com/akunzai/GSS.Authentication.CAS/releases)
## [License](LICENSE.md)