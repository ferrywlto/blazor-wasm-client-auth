# Blazor client-side authentication

## Goal
This repo is to demonstrate the missing part of most Blazor authentication scenario:
1. Display a login screen.
2. Non-authenticated user accessing routes that need authentication will redirect back to login screen.
3. Show part of the page to authorized user only.
4. Unmatched route will display "Page not found."

## Steps

### 1. Required dependencies:
- Microsoft.AspNetCore.Authorization
- Microsoft.AspNetCore.Components.Authorization
- Microsoft.AspNetCore.Components.WebAssembly.Authentication

### 2. Create `MyCustomAuthenticationStateProvider`
- Override `GetAuthenticationStateAsync` and set default user to anonymous user with no claims.
- Prepare faked signin and fake users.

### 3. Config Program.cs
- builder.Services.AddScoped<MyCustomAuthenticationStateProvider>();
- builder.Services.AddScoped<AuthenticationStateProvider>(provider => provider.GetRequiredService<MyCustomAuthenticationStateProvider>());
- builder.Services.AddAuthorizationCore();

### 4. Setup _Imports.razor
```c#
@using Microsoft.AspNetCore.Components.Authorization
@using Microsoft.AspNetCore.Authorization
// Note that the fully qualified namespace is needed. Missing the project prefix will cause Components not able to run.
@using blazor_wasm_client_auth.Pages
@attribute [Authorize]
```

### 5. Create login screen

### 6. Create RedirectToLogin component

### 7. Wrap MainLayout with `<AuthorizeView>`

### 8. Include admin only content in `NavMenu`

### 9. Wrap routing with `<AuthorizeRouteView>`

### 10. Config action for unmatched routes.



## Summary
