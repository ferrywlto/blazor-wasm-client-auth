# Blazor client-side authentication

Back to the time when I start learning authentication in Blazor, I feel frustrated as almost every tutorial are talking about creating a Blazor Server project, using Identity framework, and some CURD operations on SERVER side. Hey I know how to do authentication server side as people learning Blazor are most likely C# developers, we learn Blazor for full-stack C# opportunities so we have additional choice for creating frontend code other than React/Vue/Angular.

This is the missing piece of Blazor client-side authentication tutorial and answers the following questions:
- How to create a login screen that every user will go to this page by default?
- How to hide other screens from user access when they are not logged in?

**Note**: Client side authentication is fragile, don't depends on it. The aim of this article is not about security, it's all about UI. There are tons of tips and tricks you can find on the Internet on security hardening for both client and server side.   

## Goal
This repo is to demonstrate the Blazor client-side authentication UI flow:
1. Display a login screen.
2. Non-authenticated user accessing routes that need authentication will redirect back to login screen.
3. Show part of the page to authorized user only.
4. Unmatched route will display "Page not found." message.

## Steps

### 1. Required dependencies:
- Microsoft.AspNetCore.Authorization
- Microsoft.AspNetCore.Components.Authorization
- Microsoft.AspNetCore.Components.WebAssembly.Authentication

Let's create a Blazor wasm project. We are using .NET 6 in this project. 
```bash
dotnet new blazorwasm
dotnet add package Microsoft.AspNetCore.Authorization
dotnet add package Microsoft.AspNetCore.Components.Authorization
dotnet add package AspNetCore.Components.WebAssembly.Authentication
```

### 2. Create `MyCustomAuthenticationStateProvider`
- It must extends from `AuthenticationStateProvider`
- Override `GetAuthenticationStateAsync` and set default user to anonymous user with no claims.
- Prepare faked signin and fake users.

https://github.com/ferrywlto/blazor-wasm-client-auth/blob/master/MyCustomAuthentiationProvider.cs

This class is where all the magic happens. We will return to this later in this article.

### 3. Config Program.cs

We need to add these lines to `Program.cs`:

- builder.Services.AddScoped<MyAuthenticationStateProvider>();
- builder.Services.AddScoped<AuthenticationStateProvider>(provider => provider.GetRequiredService<MyAuthenticationStateProvider>());
- builder.Services.AddAuthorizationCore();

Caution on how we instantiate an `MyAuthenticationStateProvider` object and provide it to service container as `AuthenticationStateProvider`.
Otherwise an default `AuthenticationStateProvider` object will be used by Blazor instead of our implementation.

### 4. Setup _Imports.razor
```c#
@using Microsoft.AspNetCore.Components.Authorization
@using Microsoft.AspNetCore.Authorization
// Note that the fully qualified namespace is needed. Missing the project prefix will cause Components not able to run.
@using blazor_wasm_client_auth.Pages
@attribute [Authorize]
```

### 5. Create login screen
https://github.com/ferrywlto/blazor-wasm-client-auth/blob/4beb735aed47aeb328212c4efcdd97ef6890eaa5/Pages/Login.razor

```c#
@page "/login"
@attribute [AllowAnonymous]
```

Note that we need to place `[AllowAnonymous]` attribute here to allow user access without sign-in.

Then we have two buttons to simulate sign-in for normal user and admin user:
```c#
<button class="btn btn-primary" @onclick="OnNormalLoginClicked">Login as normal user</button>
<button class="btn btn-primary" @onclick="OnAdminLoginClicked">Login as admin</button>

private void OnNormalLoginClicked() {
    _authenticationStateProvider.FakedSignIn();
    _navigationManager.NavigateTo("/");
}
private void OnAdminLoginClicked(MouseEventArgs obj) {
    _authenticationStateProvider.FakedAdminSignIn();
    _navigationManager.NavigateTo("/");
}
```

### 6. Create RedirectToLogin component
This is a dummy component to simply redirect user back to `/login` page.

```c#
@inject NavigationManager _navigationManager

@code {
    protected override void OnInitialized() {
        _navigationManager.NavigateTo("/login");
    }
}
```
### 7. Wrap MainLayout with `<AuthorizeView>`

```c#
<AuthorizeView>
    <Authorized>
// original content of MainLayout.razor
    </Authorized>
    <NotAuthorized>
        @Body
    </NotAuthorized>
    <Authorizing>
        @Body
    </Authorizing>
</AuthorizeView>
```

### 8. Include admin only content in `NavMenu`
In `Shared/NavMenu.razor`, we will wrap the original link to `fetchData` screen with `<AuthorzieView>` tag and set the role required to see this link to "admin". We also add a button for user to logout.

```c#
<AuthorizeView Roles="admin">
// original NavLink content to fetchData
</AuthorizeView>
<AuthorizeView>
    <button class="btn btn-primary" @onclick="OnLogoutClicked">Logout</button>
</AuthorizeView>
```
### 9. Wrap routing with `<AuthorizeRouteView>`
In `App.razor`, replace `<RouteView>` tag with `<AuthorizeRouteView>` tag. Then place our `RedirectToLogin` component inside `<NotAuthorized>` tag. So whenever a user access a route that exists but not authorized yet, Blazor will redirect them back to login screen.

### 10. Config action for unmatched routes.
In `App.razor`, make sure to fill the `NotFound` tag, this is what Blazor will do when a user try to access a route that doesn't exist. If user try to access `/something-not-exist`, which is not in `['/', '/counter', '/fetchdata', '/login']`, Blazor will just shows what you have put inside the `NotFound` tag. In our case, display "Page not found".

```xml
<NotFound>
    <PageTitle>Not found</PageTitle>
    <div>Page not found.</div>
</NotFound>
```
### 11. Run and test yourself
Run and test our project by `dotnet run`.

Open browser and go to the root of our Blazor app.

Notice that although you are requesting `/`, but you are now redirected to `/login` page.

![login-page](/docs/login.png)

This is because we have these code in `App.razor`：
```c#
<NotAuthorized>
    <RedirectToLogin />
</NotAuthorized>
```

The `/` route was defined in `Index.razor`, so it fall-into `<Found>` tag. And then since we are not logged in, therefore it further fall into `<NotAuthorized>` tag and finally we landed to `<RedirectToLogin />` component. When the component initialized, it calls `_navigationManager.NavigateTo("/login");`. That's why we see the login screen. 

Now go to the address bar of our browser, type in a route that's not defined in any page. (e.g. "https://localhost:5000/something-not-exist").

Since no route matched, `App.razor` fall into `<NotFound>` tag and we will only see a message stating page is not found.

![page-not-found](/docs/not-found.png)

About `MyAuthenticationStateProvider`

This class extends `AuthenticationStateProvider` and override a method:
```c#
public override async Task<AuthenticationState> GetAuthenticationStateAsync() {
    return await Task.FromResult(new AuthenticationState(AnonymousUser));
}
```
This method will be called when user trying to access authorized page. Blazor will look for an AuthenticationState object to determine what claims the current user has and whether allow the user access specific pages.

And the `AuthenticationState` object expect an `ClaimsPrincipal` object with claims like username, roles, etc.

If an `ClaimsPrincipal` object has no claims, it is considered an anonymous user. 

This is the trick we used in `MyAuthenticationStateProvider`:
```c#
private ClaimsPrincipal AnonymousUser => new(new ClaimsIdentity(Array.Empty<Claim>()));
```

Finally, return to our login screen and try the different login button.

When you click on the normal sign-in button, you will not see the fetch data link on the navigation menu.

![nav-no-fetch](/docs/nav-no-fetch.png)

Click the logout button and we will return to login page.

Now click on the admin login button. This time we will see the fetch data link on the navigation menu.

![nav-no-fetch](/docs/nav-has-fetch.png)

This is because our faked ClaimPrinciple object has the `ClaimTypes.Role` set to "admin" and we have set the roles required to render the fetch data link to "admin":
```c#
<AuthorizeView Roles="admin">
```

## Summary
Today we learned how to create a login page and redirect all unauthorized access back to that page in Blazor. It is completely on client-side for quick UI prototyping purpose, when building your web app you should combine server-side authentication and token management logic in your `MyAuthenticationStateProvider`. That's it for now and stay tuned for my next #EverythingInCSharp article. Have a nice day! 😀
