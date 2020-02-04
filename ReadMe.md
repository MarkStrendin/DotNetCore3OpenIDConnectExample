# How to use OpenIDConnect middleware in an Asp.Net Core 3.1 web app

## Guide
I'm using https://www.youtube.com/watch?v=1zjz-aWdZHk as a guide. It's for Okta but I'm mostly concerned with  configuring the OpenIDConnect middleware. Any OpenIDConnect service should work.
Also in text form: https://developer.okta.com/blog/2019/11/15/aspnet-core-3-mvc-secure-authentication

# Steps

## Create a project
1. `mkdir OIDCTest`
2. `cd OIDCTest`
3. `dotnet new webapp`


## Set up application in identity service (AzureAD/Okta)
I'll be using AzureAD.

1. Log into Azure Portal
2. Go to **Azure Active Directory**
3. Go to **App Registrations**
4. Add a new registration
   * Redirect URI for now should be something like https://localhost:5001
      * Find/change this in Properties/launchSettings.json - the applicationUrl for https.
      * You'll add more to this list later, on Azure's end, but we'll need this to test.
   * Once created, do some initial configuration
     * **Token configuration (preview)**
       * Add Groups claim
            * `[x]` Security Groups
            * Group ID is probably fine, but I may come back to this later and change my mind
        * Add optional claim
            * Token type: ID
            * `[x]` email
            * `[x]` family_name
            * `[x]` given_name
            * `[x]` upn
        *  **Certificates & secrets**
            * New client secret
            * Name it anything (this doesn't matter for our purposes)
            * Copy the secret value and keep it for later.
                * **Do not paste this into your code**
        * **Authentication**
            * You'll want to add real urls to this page at some point
            * Logout url should look like `https://localhost:5001/signout-callback-oidc` (with a real url when you get that far)

## Add configuration for OIDC
Add to **appsettings.Development.json**, filling in values from the OIDC provider.
```json
  "OIDC": {
    "ClientId": "",
    "ClientSecret": "",
    "Authority":, ""
    "PostLogoutRedirectUri": ""
  }
 ```

For AzureAD, the AuthEndpoint should be "https://login.microsoftonline.com/XXXXXXXX" where "XXXXXXXX" is your tenant ID.

## Add nuget package for OpenIdConnect
 `dotnet add package Microsoft.AspNetCore.Authentication.OpenIdConnect --version 3.0.0`

## Add code to Startup.cs
in **ConfigureServices** method, add the following (above whatever else it's put there):
```csharp
services.AddAuthentication(options => {
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options => {
    options.SignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.Authority = Configuration["OIDC:AuthEndpoint"];
    options.RequireHttpsMetadata = true;
    options.ClientId = Configuration["OIDC:ClientId"];
    options.ClientSecret = Configuration["OIDC:ClientSecret"];
    options.ResponseType = OpenIdConnectResponseType.Code;
    options.GetClaimsFromUserInfoEndpoint = true;
    options.Scope.Add("openid");
    options.Scope.Add("profile");
    options.SaveTokens = true;
    options.TokenValidationParameters = new TokenValidationParameters {
        NameClaimType = "name",
        RoleClaimType = "groups",
        ValidateIssuer = true
    };
});
services.AddAuthorization();
```

The above code requires the following using statements
```csharp
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authentication.OpenIdConnect;
using Microsoft.IdentityModel.Protocols.OpenIdConnect;
using Microsoft.IdentityModel.Tokens;
```

## Enable authorization
In startup.cs, in the **Configure** method, add the following line

```csharp
app.UseAuthentication(); // <-- Add this line ABOVE UseAuthorization();
app.UseAuthorization(); // I should already be here
```

# Add account login/logout controller
Add **Controllers/AccountController.cs**
```csharp
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authentication.OpenIdConnect;
using Microsoft.AspNetCore.Mvc;

public class AccountController: Controller {

    public IActionResult Login() {
        if (!HttpContext.User.Identity.IsAuthenticated) {
            return Challenge(OpenIdConnectDefaults.AuthenticationScheme);
        }
        return RedirectToAction("Index", "Home");
    }

    public IActionResult Logout() {
        return new SignOutResult(new []
        {
           OpenIdConnectDefaults.AuthenticationScheme,
           CookieAuthenticationDefaults.AuthenticationScheme
        });
    }

}
```

# Create a protected page
Create a generic page to protect

Create **Pages/Protected.cshtml**
```csharp
@page
@model ProtectedModel
@{
    ViewData["Title"] = "Protected page";
}

<h1>This page should be protected</h1>
```

Create **Pages/Protected.cshtml.cs**
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.Extensions.Logging;

namespace OIDCTest.Pages
{
    public class ProtectedModel : PageModel
    {
        private readonly ILogger<IndexModel> _logger;

        public ProtectedModel(ILogger<IndexModel> logger)
        {
            _logger = logger;
        }

        public void OnGet()
        {

        }
    }
}
```

Protect the page in **Startup.cs** by adding onto the AddRazorPages() call

```csharp
services.AddRazorPages()
    .AddRazorPagesOptions(options => {
        options.Conventions.AuthorizePage("/Protected");
    });
```
