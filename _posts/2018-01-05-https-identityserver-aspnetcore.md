---
title: HTTPS in IdentityServer4 and ASP.NET Core 2
tags: c# .net asp.net.core security identityserver
header:
  image: "/assets/2018/01-05/header1280.jpg"
  teaser: "/assets/2018/01-05/header1280.jpg"

# image paths:
#   publish:      (/assets/2018/mm-dd/pic.jpg)
#   edit _drafts: (/assets/2018/01-05/pic.jpg)
#   edit _post:   (../assets/2018/mm-dd/pic.jpg)
---

This post will examine how to enable SSL for localhost and how to use it with IdentityServer4 and an ASP.NET Core 2 client.

<!--more-->

This article adds HTTPS support to the projects created in an earlier post, [IdentityServer4 Without Entity Framework]({{ site.baseurl }}{% post_url 2018-01-02-identityserver4-without-entity-framework %}), using the certificates generated by [the first part]({{ site.baseurl }}{% post_url 2018-01-04-localhost-ssl-identityserver-certificates %}) of this two-part series. The files which are changed can be found in [this](https://github.com/MV10/IdentityServer4.AspNetCore2.Https) repository.

## X.509 Certificate Helper Class

In the previous post, we built several Powershell scripts to help us generate X.509 certificates used by SSL and IdentityServer4 tokens. Those certificates are stored in the Windows certificate store, so let's build a simple helper-class to retrieve them. Create a new class named `X509Helper.cs` in either the client web app project or the IdentityServer4 project, put the following code into it, and copy the completed class file to the other project.

```
using System.Security.Cryptography.X509Certificates;

namespace X509Helper
{
    public static class X509
    {
        public static X509Certificate2 GetCertificate(string thumbprint)
        {
            using(X509Store certStore = new X509Store(StoreName.My, StoreLocation.CurrentUser))
            {
                certStore.Open(OpenFlags.ReadOnly);
                X509Certificate2Collection certCollection = certStore.Certificates.Find(X509FindType.FindByThumbprint, thumbprint, false);
                if(certCollection.Count > 0) return certCollection[0];
            }
            return null;
        }
    }
}
```

## Retrieve the Thumbprints

Open a Powershell console, navigate to the folder you created during the previous post, and execute the `cert_list_thumbprints` script we created.

![Certthumbs](/assets/2018/01-05/certthumbs.png)

Leave the window open for now, we'll come back to it shortly.

## Choose Kestrel Hosting

This time, we'll also run the client web application under Kestrel hosting. If you enable SSL with IIS Express, it will create a new localhost certificate and will force you to use the wrong port number. Kestrel configuration is much more flexible. 

For both web apps, right-click the project name, open Properties, go to the Build tab, and change the Profile field to reflect the project name, not IIS Express.

![Choosekestrel](/assets/2018/01-05/choosekestrel.png)

The Build tab's App URL and port doesn't matter, it will be overriden by changes we'll add to the Kestrel start-up code in the next section.

**Important:** Before you run either web app, check the Visual Studio toolbar to confirm Kestrel is selected. It should show the project name, not "IIS Express." Sometimes Visual Studio randomly switches back to IIS Express.
{: .notice--warning}

![Toolbar](/assets/2018/01-05/toolbar.png)

## Kestrel Configuration

A Kestrel web host runs as a console program. Like any console program, it has a `Program.cs` with a `static Main` method. Open this file for the IdentityServer4 project. Add the two `using` statements shown below.

```
using System.Net;
using X509Helper;
```

Next, locate the `.UseStartup<Startup>()` command and insert the following code after that, but before the `.Build();` command. Copy the thumbprint for the localhost certificate from your Powershell console and paste it into the `GetCertificate` call where it says `THUMBPRINT` in the code below.

```
.UseKestrel(options =>
{
    options.Listen(IPAddress.Any, 5000, listenOptions =>
    {
        listenOptions.UseHttps(X509.GetCertificate("THUMBPRINT")); // localhost.crt thumbprint
    });
})
```

The `Listen` call specifies port `5000`. Make the same two changes to `Program.cs` in the client web app, but change the port number to `5002`.

The `IpAddress` could also be set to `Loopback` which is the same as localhost (127.0.0.1), but using `Any` means your code works equally well in development or test without changes (assuming you're also OK with the port assignment -- which could just as easily be a configuration item).

## IdentityServer4 Startup Configuration

The changes to ASP.NET Core configuration are a bit more extensive, and IdentityServer4 has several requirements that don't apply to a separate client application. Open `Startup.cs` in the IdentityServer project and add the following `using` statements.

```
using X509Helper;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Rewrite;
```

Next, alter the beginning of the `ConfigureServices` method as shown in the partial code block below. Copy your token-signing and token-validation thumbprints from your Powershell window into the `GetCertificate` calls as indicated by the `THUMBPRINT` placeholders.

```
public void ConfigureServices(IServiceCollection services)
{
    // change this
    services.AddMvc(options =>
    {
        options.SslPort = 5000;
        options.Filters.Add(new RequireHttpsAttribute());
    });

    // add this
    services.AddAntiforgery(options =>
    {
        options.Cookie.Name = "_af";
        options.Cookie.HttpOnly = true;
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        options.HeaderName = "X-XSRF-TOKEN";
    });

    services.AddIdentityServer()

        // remove this
        // .AddDeveloperSigningCredential()

        // add these
        .AddSigningCredential(X509.GetCertificate("THUMBPRINT"))   // signing.crt thumbprint
        .AddValidationKey(X509.GetCertificate("THUMBPRINT"))       // validation.crt thumbprint

        .AddInMemoryClients(ConfigureIdentityServer.GetClients())
        .AddInMemoryIdentityResources(ConfigureIdentityServer.GetIdentityResources())
        .AddProfileService<UserProfileService>();

    ... // (additional code not shown here)
```

There is one more line of code to add to the `Configure` method immediately before the call to `UseIdentityServer`.

```
// add this
app.UseRewriter(new RewriteOptions().AddRedirectToHttps());

app.UseIdentityServer(); // includes a call to UseAuthentication
```

## Google Sign-In

Since we haven't added a way to register local user accounts for username/password logins, we'll test the code again using Google's third-party OAuth2 login. Our earlier non-SSL projects borrowed the Google `ClientId` and `ClientSecret` shared in the IdentityServer documentation, which is associated with `http://localhost`. We need to change that to a Google app service associated with an `https` URI. I have registered one which anyone can use for demo and testing purposes.

At the end of the `ConfigureServices` method, change the Google identifiers to the following.

```
services.AddAuthentication()
    .AddGoogle("Google", options =>
    {
        options.SignInScheme = IdentityServerConstants.ExternalCookieAuthenticationScheme;

        // these are for https://localhost
        options.ClientId = "1027693607760-g1i92hf92sf2a4htufesmt0e8sfsurt9.apps.googleusercontent.com";
        options.ClientSecret = "C6GebLkVXKPZmr1pMB26rUUi";
    });
```


## Client App Startup Configuration

Open `Startup.cs` for the client web app and add these three `using` statements.

```
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Rewrite;
```

We'll make similar changes to the start of `ConfigureServices` but notice we reference port 5002 this time instead of port 5000.

```
        public void ConfigureServices(IServiceCollection services)
        {
            // change this
            services.AddMvc(options =>
            {
                options.SslPort = 5002;
                options.Filters.Add(new RequireHttpsAttribute());
            });

            // add this
            services.AddAntiforgery(options =>
            {
                options.Cookie.Name = "_af";
                options.Cookie.HttpOnly = true;
                options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
                options.HeaderName = "X-XSRF-TOKEN";
            });

            ... // (additional code not shown here)
```

There is one more change further down in `ConfigureServices`. Find the the OpenID Connect configuration and change the `Authority` to connect to IdentityServer4 using HTTPS:

```
options.Authority = "https://localhost:5000";
```

The client version of the `Configure` method gets the same HTTPS-rewrite setting we added to IdentityServer, but this time insert it before the `UseAuthentication` call.

```
// add this
app.UseRewriter(new RewriteOptions().AddRedirectToHttps());

app.UseAuthentication();
```


## IdentityServer4 Client URIs

Next we must alter the URIs that define IdentityServer's communications back to the client. Open the `ConfigureIdentityServer.cs` file and change the three HTTP references to HTTPS in the `GetClients` method.

![Clienturis](/assets/2018/01-05/clienturis.png)

## A Note About Logout

You may recall that the client application begins the logout process using an `<a>` anchor-tag (styled as a button) that issues an `HTTP GET` to a handler method in the page model. However, the URL on the anchor tag is a relative `href` as shown below, so it doesn't need to be modified to use HTTPS.

```
<a class="btn btn-default" href="/About?handler=logoff">Logoff</a>
```

## Test-Run

**Remember:** Before you run either web app, check the Visual Studio toolbar to confirm Kestrel is selected. It should show the project name, not "IIS Express." Sometimes Visual Studio randomly switches back to IIS Express.
{: .notice--warning}

![Toolbar](/assets/2018/01-05/toolbar.png)

Select the IdentityServer4 project, confirm it will use Kestrel hosting, and press CTRL+F5 to launch the server. You can see that our Kestrel startup code overrides the App URL on the project properties Build tab. You'll see the same message in the client app's Kestrel console window.

![Kestrelconsole](/assets/2018/01-05/kestrelconsole.png)

Next, select the client web app project, confirm it will use Kestrel hosting, and launch the application. Finally, run through the login process -- remember, the About page is the protected resource that triggers authentication. Confirm that Google third-party login works end-to-end. You can revisit the non-HTTPS screenshots at the end of the [original article]({{ site.baseurl }}{% post_url 2018-01-02-identityserver4-without-entity-framework %}).

## Conclusion

As you can see, it's relatively easy to switch to full-time SSL while avoiding browser nagging. The only other point I'd like to make is that Kestrel is not intended to run by itself. For production usage, you'll often proxy requests through another web server (typically IIS or Apache, although other parts of my examples are Windows-specific, particularly the certificate store). In that case, the proxy would handle SSL and proxy-to-Kestrel communications would be internal over HTTP. The details are explained by Microsoft [here](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers/?tabs=aspnetcore2x).

![Kestreldocs](/assets/2018/01-05/kestreldocs.png)