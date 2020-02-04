# Guide
I'm using https://www.youtube.com/watch?v=1zjz-aWdZHk as a guide. It's for Okta but I'm mostly concerned with  configuring the OpenIDConnect middleware. Any OpenIDConnect service should work.

# Steps

## Set up project
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

## Add configuration for OIDC
Add to **appsettings.Development.json**, filling in values from the OIDC provider.
```
  "OIDC": {
    "ClientId": "",
    "ClientSecret": "",
    "Domain": "",
    "PostLogoutRedirectUri": "" 
  }
 ```

 ## Add nuget package for OpenIdConnect
 `dotnet add package Microsoft.AspNetCore.Authentication.OpenIdConnect --version 3.0.0`


