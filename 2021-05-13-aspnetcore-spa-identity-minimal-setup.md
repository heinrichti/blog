---
title: "AspNetCore, SPA, Identity: Minimal Setup"
date: "2021-05-13"
categories: 
  - "programming"
tags: 
  - "net"
  - "asp-net-core"
  - "c"
  - "vue-js"
---

Recently I tried to setup a very basic thing: An API backend based on ASP.NET Core and a HTML/JavaScript single page application as frontend (in this case with Vue.js). I should also be able to register an account with username/password or with an external login like Google. Every request to the API should only be accessible if the user is logged in (except the login/register page of cours). I decided against EntityFramework in favor of Dapper, which brought a few challanges along.

The first steps seemed simple and promising, just create a new project from the template "ASP.Net Core Project with Angular" with individual Accounts, and see for yourself what you get: An Angular app with an API backend, a register page, account management and so on. But here is the thing: The register/login/account management is not part of the client app! This may be just me, but it just doesn't feel right. I am using transitions between pages in my client app, but when switching to the login I just get a full page reload. Also the layout of the register/login pages was very different from what I had in mind. So how to change that? I found no easy way, so here just a short description of what I want to achieve: The register/login/account management is part of the front-end app and just communicates with the backend instead of being implemented as a backend app as razor pages. Next up is EntityFramework. The default template seems to force me to use EF to store my user accounts. Furthermore it uses IdentityServer 4, which I do not know what it does but where at the time of writing the website is down and IdentityServer 5 seems to be purely commercial, so maybe I should throw that out as well as EntityFramework. Why all this technical overhead when I want a simple and elegant solution? At this point I was already pretty frustrated as nothing was working as expected. If you feel the same way, read on.

After it did not work to customize the default template to my needs, I startet thinking: What is \*actually\* required to make this work? Let´s instead create a minimal web api project and go from there, add all the features we need as we go. Create a simple "ASP.NET Core Web API" Project with "Authentication Type: None".

From here out API is already working, so let´s get started with the SPA: First create your frontend in the ClientApp folder (for Vue.js this would be something like `vue create myapp`), just make sure that its root is in the colder ClientApp and not a subfolder. Add a reference to `Microsoft.AspNetCore.SpaServices.Extensions` and add these lines of code to your `Startup.cs`.

public void Configure(IApplicationBuilder app)
{
  // ... important that this code comes last in the Configure method
  app.UseSpa(config =>
  {
    config.Options.SourcePath = "ClientApp";
#if DEBUG
    config.UseProxyToSpaDevelopmentServer("https://127.0.0.1:8080");
#endif
  });
}

There are a few caveats already, the UseSpa is like a catch-all route. So if you register your controllers after this they will never be hit. Also note that you have to configure your client-app (webpack dev server) to run at port 8080 (change this to whatever you like) with HTTPS and that the certificate has to be trusted. Note that I use the IP instead of the DNS name as the webpack dev server certificate will be generated for the IP and not the hostname. If it is not HTTPS or the certificate is not trusted, then hot reloading of your frontend will not work. For Vue.js you could create a vue.config.js like this:

module.exports = {
 configureWebpack: {
     devServer: {
      https: true,
      host: '127.0.0.1'
     }
 }
};

Alright, now our SPA should be working fine! Just make sure to have it started with `npm run serve` so that our development proxy can connect to it.

Next up is CSRF or cross site request forgery. If you are not familiar with CSRF, the problem is basically that your cookie for this site is transferred with each and every request to the site. And as our authentication will use this cookie to authenticate the user if someone places a form somewhere which does e.g. a "delete this user account" post to our site, the cookie is transferred, the user therefore authenticated and the account deleted, without the user knowing how this could happen. The solution for this is to add a token to the request that is generated when the user visits the page, the CSRF-Token. You want this, and you want it either on each and every form/POST/PUT/... or maybe even on each and every request.

So there are two problems here we need to solve: Validating CSRF-Tokens and transferring the token into the client app. Back to our `Startup.cs`:

public void ConfigureServices(IServiceCollection services)
{
...
  // Required to register the CSRF-Validator
  services.AddMvcCore().AddViews();
  // Treat the header X-CSRF-TOKEN as anti forgery token to validate
  // We will use this header when we make requests via JavaScript
  services.AddAntiforgery(options => options.HeaderName = "X-CSRF-TOKEN");
}

public void Configure(IApplicationBuilder app, IAntiforgery antiforgery)
{
...
  // this registers a middleware that provides an CSRF antiforgery-token in a cookie that is accessible to our client app
  app.Use(next => context =>
  {
    string path = context.Request.Path.Value;

    if (string.Equals(path, "/", StringComparison.OrdinalIgnoreCase) ||
                    string.Equals(path, "/index.html", StringComparison.OrdinalIgnoreCase))
    {
      var tokens = antiforgery.GetAndStoreTokens(context);
      // notice the HttpOnly=false, this allows us to access this cookie from javascript
      context.Response.Cookies.Append("XSRF-TOKEN", tokens.RequestToken, new CookieOptions() { HttpOnly = false });
    }

     return next(context);
   });
...
}

Great, now we get our antiforgery token delivered to our client app, but how do we use it? Well, "simply" embed the token in your forms or add the header "X-CSRF-TOKEN" to your requests. For your requests in JavaScript it might look something like this:

function getAntiForgeryToken() {
  // Parsing our previously generated cookie
  var name = 'XSRF-TOKEN' + "=";
  var decodedCookie = decodeURIComponent(document.cookie);
  var ca = decodedCookie.split(';');
  for (var i = 0; i < ca.length; i++) {
      var c = ca\[i\];
      while (c.charAt(0) === ' ') {
          c = c.substring(1);
      }
      if (c.indexOf(name) === 0) {
          return c.substring(name.length, c.length);
      }
  }
  return "";
}

var antiForgeryToken = getAntiForgeryToken();
// adds the antoforgery header used to validate the request
var options = {
  'X-CSRF-TOKEN': antiForgeryToken
}
var response = await fetch('/test', options);

Now for forms I wrote a little vue-component that generates a form including the antiforgery token:

<template>
  <form :action="action" :method="method">
    <slot></slot>
    <input name="\_\_RequestVerificationToken" type="hidden" :value="antiForgeryToken" />
  </form>
</template>

<script>
import fetch from '../plugins/fetch.js'

export default {
  name: 'vue-form',
  props: {
    action: String,
    method: String
  },
  computed: {
    antiForgeryToken() {
      return fetch.getAntiForgeryToken();
    }
  }
};
</script>

I think all that´s left is the authentication. There is a lot to unpack when looking at this. First you will need an IdentityUser, which represents your logged in users:

public class ApplicationUser : IdentityUser
{
  public string LoginProvider { get; set; }
  public string ProviderKey { get; set; }
  public string ProviderDisplayName { get; set; }
}

In this case I already added a few fields that we will need later when registering with OAuth2 providers. In my case I do not need roles at the moment, so my Startup.cs could contain something like this:

public void ConfigureServices(IServiceCollection services)
{
  services.AddIdentityCore<ApplicationUser>(options =>
  {
    // set the defaults to your liking
    options.Password.RequireDigit = false;
    options.Password.RequiredLength = 12;
    options.Password.RequireLowercase = false;
    options.Password.RequireNonAlphanumeric = false;
    options.Password.RequireUppercase = false;
  })
    .AddUserStore<ApplicationUserStore>()
    .AddSignInManager<SignInManager<ApplicationUser>>()
    .AddDefaultTokenProviders();

  services.AddAuthentication(IdentityConstants.ApplicationScheme)
    .AddCookie(IdentityConstants.ApplicationScheme, options =>
    {
      options.Events.OnRedirectToLogin += context =>
      {
        context.Response.StatusCode = 401;
        return Task.CompletedTask;
      };
    })
    .AddCookie(IdentityConstants.ExternalScheme)
    .AddCookie(IdentityConstants.TwoFactorUserIdScheme);
    .AddGoogle(options =>
    {
      options.ClientId = Configuration\["Authentication:Google:ClientId"\];
      options.ClientSecret = Configuration\["Authentication:Google:ClientSecret"\];
      options.SignInScheme = IdentityConstants.ExternalScheme;
    });
...
}

public void Configure(...)
{
  ...
  app.UseAuthentication();
  app.UseAuthorization();
  ...
}

Wow, this is quite some code there... But let´s take it one line at a time, a little scary but not too scary. Basically we are configuring our authentication here. `AddIdentityCore` registers the services required for this to work at all. `AddUserStore<ApplicationUserStore>()` is required as we are not using EF Core. Somehow we need to store user-stuff, and that is handled by our ApplicationUserStore, which is just a class implementing `IUserStore<ApplicationUser>`. Depending on what you will do with it you will also have to implement `IUserPasswordStore<>`, `IUserEmailStore<>` and `IUserLoginStore<>`, but the app will tell you with an exception if anything is missing, so no need to go into detail here. The `AddSignInManager<SignInManager<ApplicationUser>>()` will register a SignInManager, which you can use to log users in/out (more to that later).

Now about the next part gave me some trouble, basically because all the tutorials I found want you to register the default CookieScheme in `services.AddAuthentitcation`. All I can say is that it will work this way, with a cookie, but when you change it you will get issues when trying to log your users in/out, or when integrating external services like Google. The same for the next AddCookie-lines, you just need them for OAuth2 or logging users in.

The AddGoogle line is just specifying that you support authentication through Google. You can find many resources online how to make this work, just know that you will have to register your app in the google developer console or this will not work at all.

Funny thing, now we have all the pieces together, maybe you even managed to get this far, but nobody ever told you how to actually USE this! For this I strongly recommend reading the source code of Microsoft.AspNetCore.Identity.Ui [here](https://github.com/aspnet/Identity/tree/master/src/UI/Areas/Identity/Pages/V4/Account). This is where our previously registered LogInManager comes into play. Let´s have a look at the register controller, I will leave the LoginController to you:

\[ApiController\]
\[Route("\[controller\]")\]
// automatically validates CSRF-token on POST, PUT etc. but not GET
\[AutoValidateAntiforgeryToken\]
public class RegisterController : ControllerBase
{
    private readonly IUserStore<ApplicationUser> \_userStore;
    private readonly UserManager<ApplicationUser> \_userManager;
    private readonly SignInManager<ApplicationUser> \_signInManager;

    public RegisterController(IUserStore<ApplicationUser> userStore,
        UserManager<ApplicationUser> userManager,
        SignInManager<ApplicationUser> signInManager)
    {
        // our own store using dapper
        \_userStore = userStore;
        // usermanager can create an delete users
        \_userManager = userManager;
        // SignInManager can sign users in, in our case give them a cookie
        \_signInManager = signInManager;
    }

    \[HttpPost\]
    public async Task<IActionResult> Index(\[FromForm\] RegisterModel model)
    {
        var user = new ApplicationUser();
        var emailStore = (IUserEmailStore<ApplicationUser>)\_userStore;
        await \_userStore.SetUserNameAsync(user, model.Name, CancellationToken.None);
        await emailStore.SetEmailAsync(user, model.Email, CancellationToken.None);

        var result = await \_userManager.CreateAsync(user, model.Password);
        if (!result.Succeeded)
            return Unauthorized();

        await \_signInManager.SignInAsync(user, true);
        return LocalRedirect("/");
    }

    \[HttpPost\]
    \[Route("External")\]
    public IActionResult External(\[FromForm\] string provider)
    {
        var properties = \_signInManager.ConfigureExternalAuthenticationProperties(provider, "/Register/External");
        return new ChallengeResult(provider, properties);
    }

    \[HttpGet\]
    \[Route("External")\]
    public async Task<IActionResult> External(string returnUrl = null, string remoteError = null)
    {
        var info = await \_signInManager.GetExternalLoginInfoAsync();

        if (info == null)
            return Unauthorized();

        var loginStore = (IUserLoginStore<ApplicationUser>)\_userStore;
        var emailStore = (IUserEmailStore<ApplicationUser>)\_userStore;
        var user = new ApplicationUser();
        await \_userStore.SetUserNameAsync(user, info.Principal.Identity.Name, CancellationToken.None);
        await emailStore.SetEmailAsync(user, info.Principal.Claims.First(x => x.Type == ClaimTypes.Email).Value, CancellationToken.None);
        user.LoginProvider = info.LoginProvider;
        user.ProviderKey = info.ProviderKey;
        user.ProviderDisplayName = info.ProviderDisplayName;

        await \_userStore.CreateAsync(user, CancellationToken.None);

        var result = await \_signInManager.ExternalLoginSignInAsync(info.LoginProvider, info.ProviderKey, isPersistent: false, bypassTwoFactor: true);
        if (!result.Succeeded)
            return Unauthorized();

        return LocalRedirect("/#/");
    }
}

I think the Index action is pretty self-explaining. Create a user and log the user in. Then redirect to home. The external action is a little more interesting. Here we make use of our previously configured authentication provider ("Google") and challenge it. This challenge will result in a redirect to the external google login page, which then redirects back to us and we will handle the result in the action GET External. The code should work with all providers, we just implemented it only for Google. To make this maybe even more clear how to implement this, here the Vue-code for the external login:

  <v-form id="external-account" method="post"
    action="/Login/External">
    <button class="btn btn-primary"
      name="provider" value="Google">Google</button>
  </v-form>

All this is in my opinion the minimal setup for using ASP.NET Core with an SPA. I fear that most of this is necessary in most projects of this kind. Yes, you can save a little work in using EntityFramework and the default generated Auth-Pages, but I really don't like the way how the app feels when redirecting to another page for the login, and effectively having the layout in two places, the client app and the razor pages.
