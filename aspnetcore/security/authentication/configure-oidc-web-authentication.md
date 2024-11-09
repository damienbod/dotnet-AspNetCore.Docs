---
title: Configure OpenID Connect Web (UI) authentication in ASP.NET Core
author: damienbod
description: Learn how to set up OpenID Connect authentication in an ASP.NET Core app.
monikerRange: '>= aspnetcore-8.0'
ms.author: riande
ms.custom: mvc
ms.date: 04/02/2024
uid: security/authentication/configure-oidc-web-authentication
---
# Configure OpenID Connect Web (UI) authentication in ASP.NET Core

By [Damien Bowden](https://github.com/damienbod)

[View or download sample code](https://github.com/dotnet/AspNetCore.Docs/tree/main/aspnetcore/security/authentication/configure-oidc-web-authentication/sample/oidc-net8)

This article covers the following areas:

* What is an OpenID Connect confidential interactive client
* Create an OpenID Connect client in ASP.NET Core
* Examples of OpenID Connect client with code snippets
* Using third party OpenID Connect provider clients
* Backend for frontend (BFF) security architecture
* Advanced features, standards, extending the an OpenID Connect client

## What is an OpenID Connect confidential interactive client

[OpenID Connect](https://openid.net/developers/how-connect-works/) can be used to implement authentication in ASP.NET Core applications. The recommended way is to use an OpenID Connect confidential client using the code flow. Using the [Proof Key for Code Exchange by OAuth Public Clients (PKCE)](https://datatracker.ietf.org/doc/html/rfc7636) is recommended for this implementation. Both the application client and the user of the application are authenticated in the confidential flow. The application client uses a client secret or a client assertion to authenticate. 

Public OpenID Connect/OAuth clients are no longer recommended for web applications.

The default flow works as shown in the following diagram:

![OIDC code flow confidential client using PKCE](~/security/authentication/configure-oidc-web-authentication/_static/oidc-confidential-pkce-flow-drawio.png)

OpenID Connect comes in many variations and all server implementations have slightly different parameters and requirements. Some servers don’t support the user info endpoint, some still don’t support PKCE and others require special parameters in the token request. Client assertions can be used instead of client secrets. New standards also exist which add extra security on top of the OpenID Connect Core, for example FAPI, CIBA or DPoP for downstream APIs.

> [!NOTE]
> From .NET 9, [OAuth 2.0 Pushed Authorization Requests (PAR) RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126) is used per default, if the OpenID Connect server supports this. This is a three step flow and not a two step flow as shown above. (User Info request is an optional step.)

## Create an Open ID Connect code flow client using Razor Pages

The following section shows how to implement an OpenID Connect client in an empty ASP.NET Core Razor page project. The same logic can be applied to any ASP.NET Core web project with only the UI integration being different.

### Add OpenID Connect support

Add the [Microsoft.AspNetCore.Authentication.OpenIdConnect](https://www.nuget.org/packages/Microsoft.AspNetCore.Authentication.OpenIdConnect) Nuget packages to the ASP.NET Core project.

### Setup the OpenID Connect client

Add the authentication to the web application using the builder.Services in the **Program.cs** file. The configuration is dependent on the OpenID Connect server. Each OpenID Connect server requires small differences in the setup.

```csharp
builder.Services.AddAuthentication(options =>
{
	options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
	options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options =>
{
	configuration.GetSection("OpenIDConnectSettings").Bind(options);
	
	options.Authority = configuration["OpenIDConnectSettings:Authority"];
	options.ClientId = configuration["OpenIDConnectSettings:ClientId"];
	options.ClientSecret = configuration["OpenIDConnectSettings:ClientSecret"];

	options.SignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
	options.ResponseType = OpenIdConnectResponseType.Code;

	options.SaveTokens = true;
	options.GetClaimsFromUserInfoEndpoint = true;

	options.MapInboundClaims = false;
	options.TokenValidationParameters.NameClaimType = JwtRegisteredClaimNames.Name;
	options.TokenValidationParameters.RoleClaimType = "roles";
});
```

See [Secure an ASP.NET Core Blazor Web App with OpenID Connect (OIDC)](xref:blazor/security/blazor-web-app-oidc) for details on the different OpenID Connect options.

See [Mapping, customizing, and transforming claims in ASP.NET Core](xref:security/authentication/claims) for the different claims mapping possibilities.

> [!NOTE]
> The following namespaces are required:

```csharp
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authentication.OpenIdConnect;
using Microsoft.IdentityModel.Protocols.OpenIdConnect;
using Microsoft.IdentityModel.Tokens;
```

### Setup the configuration properties

Add the OpenID Connect client settings to the application configuration properties. The settings must match the client configuration in the OpenID Connect server. No secrets are persisted in the application settings, the secrets are stored in a Key Vault in production environments or in user secrets in a development environment.

```json
"OpenIDConnectSettings": {
  // OpenID Connect URL
  "Authority": "https://localhost:44318",
  // client ID from the OpenID Connect server
  "ClientId": "oidc-pkce-confidential",
  //"ClientSecret": "--stored-in-user-secrets-or-key-vault--"
},
```

### Update the ASP.NET Core pipeline method in the program class.

The UseRouting must be implemented before the UseAuthorization method.

```csharp
app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseRouting();
// Authorization is required for middleware 
// after the UseAuthorization method
app.UseAuthorization();
app.MapRazorPages();
```

### Force authorization

Add the Authorize attribute to the protected razor pages, for example the Index.cshtml.cs file

```csharp
[Authorize]
```

A better way would be to force the whole application to be authorized and opt out for unsecure pages

```csharp
builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder()
    .RequireAuthenticatedUser()
    .Build());
```

### Add a new Logout.cshtml and SignedOut.cshtml Razor page to the project

A logout is required to sign out both the cookie session and the OpenID Connect session. The whole application needs to redirect to the OpenID Connect server to sign out. After a successful sign-out, the application will open the RedirectUri route.

Implement a default sign out page and change the Logout razor page code with this:

```csharp
[Authorize]
public class LogoutModel : PageModel
{
    public IActionResult OnGetAsync()
    {
        return SignOut(new AuthenticationProperties
        {
            RedirectUri = "/SignedOut"
        },
        CookieAuthenticationDefaults.AuthenticationScheme,
        OpenIdConnectDefaults.AuthenticationScheme);
    }
}
```

The `SignedOut.cshtml` requires the AllowAnonymous attribute.

```csharp
[AllowAnonymous]
public class SignedOutModel : PageModel
{
    public void OnGet()
    {
    }
}
```

### Add a login, logout button for the user.

```
@if (Context.User.Identity!.IsAuthenticated)
{
	<li class="nav-item">
		<a class="nav-link text-dark" asp-area="" asp-page="/Logout">Logout</a>
	</li>

	<span class="nav-link text-dark">Hi @Context.User.Identity.Name</span>
}
else
{
	<li class="nav-item">
		<a class="nav-link text-dark" asp-area="" asp-page="/Index">Login</a>
	</li>
}
```

## Examples with code snippets

### Example Using User Info endpoint

The OpenID Connect options can be used to map claims, implement handlers or even save the tokens in the session for later usage. 

The **Scope** option can be used to request different claims or a refresh token which is sent as information to the OpenID Connect server. Requesting the **offline_access** is asking the server to return a reference token which can be used to refresh the session without authenticating the user of the application again.

```csharp
services.AddAuthentication(options =>
{
	options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
	options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(OpenIdConnectDefaults.AuthenticationScheme, options =>
{
	options.Authority = authConfiguration["IdentityProviderUrl"];
	options.ClientSecret = authConfiguration["ClientSecret"];
	options.ClientId = authConfiguration["Audience"];
	options.ResponseType = OpenIdConnectResponseType.Code;

	options.Scope.Clear();
	options.Scope.Add("openid");
	options.Scope.Add("profile");
	options.Scope.Add("email");
	options.Scope.Add("offline_access");

	options.ClaimActions.Remove("amr");
	options.ClaimActions.MapJsonKey("website", "website");

	options.GetClaimsFromUserInfoEndpoint = true;
	options.SaveTokens = true;

    // .NET 9 feature
	options.PushedAuthorizationBehavior = PushedAuthorizationBehavior.Require;

	options.TokenValidationParameters = new TokenValidationParameters
	{
		NameClaimType = JwtClaimTypes.Name,
		RoleClaimType = JwtClaimTypes.Role,
	};
});
```

See the following code example: 

https://github.com/damienbod/keycloak-backchannel/tree/main/RazorPagePar

### Implementing Microsoft identity providers

Microsoft has multiple identity providers and OpenID Connect implementations. Microsoft has different OpenID Connect servers:

* Microsoft Entra ID
* Microsoft Entra External ID
* Azure AD B2C

If authenticating using one of the Microsoft identity providers in ASP.NET Core, it is recommended to use the [Microsoft.Identity.Web](https://github.com/AzureAD/microsoft-identity-web) Nuget packages.

The Microsoft.Identity.Web Nuget packages is a Microsoft specific client built on top on the ASP.NET Core OpenID Connect client with some changes to the default client.

## Using third party OpenID Connect provider clients

Many OpenID Connect server implementations create Nuget packages which are optimized for the same OpenID Connect implementation. These packages implement the OpenID Connect client specifics with the extras required by the specific OpenID Connect server. Microsoft.Identity.Web is one example of this. 

If implementing multiple OpenID Connect clients from different OpenID Connect servers in a single application, it is normally better to revert to the default ASP.NET Core implementation as the different clients overwrite some options which affect the other clients. 

[OpenIddict Web providers](https://documentation.openiddict.com/integrations/web-providers) is a client implementation which supports many different server implementations.

[IdentityModel](https://github.com/IdentityModel/IdentityModel) is a .NET standard helper library for claims-based identity, OAuth 2.0 and OpenID Connect. This can also be used to help with the client implementation.

## Backend for frontend (BFF) security architecture

It is no longer recommended to implement OpenID Connect public clients for any web applications. 

See the [draft OAuth 2.0 for Browser-Based Applications](https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps/) for further details.

If implementing **web** applications which have no backend, it is recommended to use the [Backend for Frontend (BFF) pattern](/azure/architecture/patterns/backends-for-frontends) security architecture.  This pattern can be implemented in different ways, but the authentication is always implemented in the backend and no sensitive data is sent to the web client for further authorization or authentication flows.

## Advanced features, standards, extending the OIDC client

### Logging

Debugging OpenID Connect clients can be hard. Personally identifiable information (PII) data is not logged by default. If debugging in development mode, the ** IdentityModelEventSource.ShowPII** can be used to log sensitive personal data. This should never by deployed to productive servers.

```csharp
//using ...

using Microsoft.IdentityModel.Logging;

var builder = WebApplication.CreateBuilder(args);

//... code 

var app = builder.Build();

IdentityModelEventSource.ShowPII = true;

//... code 

app.Run();
```

### OIDC and OAuth Parameter Customization

The OAuth and OIDC authentication handlers  [`AdditionalAuthorizationParameters`](https://source.dot.net/#Microsoft.AspNetCore.Authentication.OAuth/OAuthOptions.cs,ddb988460467cfbf) option allows customization of authorization message parameters that are usually included as part of the redirect query string:

## Map claims from OpenID Connect

Refer to the following document:

[Mapping, customizing, and transforming claims in ASP.NET Core](xref:security/authentication/claims)

## Blazor OpenID Connect

Refer to the following document:

[Secure an ASP.NET Core Blazor Web App with OpenID Connect (OIDC)](xref:blazor/security/blazor-web-app-oidc)

## Standards

[OpenID Connect 1.0](https://openid.net/specs/openid-connect-core-1_0-final.html)

[Proof Key for Code Exchange by OAuth Public Clients](https://datatracker.ietf.org/doc/html/rfc7636)

[The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)

[OAuth 2.0 Pushed Authorization Requests (PAR) RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126)