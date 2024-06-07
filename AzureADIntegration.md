# Azure Active Directory in ASP .NET Core

## A guide to integrating Azure Active Directory authentication within an ASP .NET Core with MVC Web Application

This is not meant to be a comprehensive guide.  For more information on integrating Azure with .NET applications, [Microsoft provides a plethora of information](https://learn.microsoft.com/en-us/dotnet/azure/?utm_source=aspnet-start-page&utm_campaign=vside) on the subject on their own web domain.

This is a guide to integrating Azure Active Directory (AD) authentication within an ASP .NET Core web application with MVC (Model-View-Controller), using .NET Framework 6.0.  This guide also assumes you're using the Program.cs file in minimal API style, and in absence of a Startup.cs file.

> [!NOTE]
> "Azure AD" may also be known as "MS Entra ID", or "Microsoft Entra ID".  The terms can be used interchangeably, but for the purposes of this documentation, "Azure AD" will be used, as this document will not be referencing Windows Server Active Directory in any way, and so confusion should be minimal.  Microsoft has [provided an explanation](https://learn.microsoft.com/en-us/entra/fundamentals/new-name) for the change in name of this service.

### ASP .NET Core Web Application

You'll need to have a web application in ASP .NET Core to connect with Azure AD.  This guide assumes such a web application already exists, and details how to add the necessary components for on-premise Azure AD authentication.

You can also create a new web application project, and configure it to connect with Microsoft identity platform.  The steps for doing this in Visual Studio 2022 are highlighted as follows:

- Open Visual Studio 2022.
- Select **Create a new project**.
- Scroll down, or search for, and select "ASP.NET Core Web App (Model-View-Controller)".
- Click **Next**.
- Under 'Configure your new project,' enter a Project name and Solution name.  Click **Next**.
- For Framework, you can choose '.NET 6.0 (Long Term Support).'
- Next, select 'Microsoft identity platform' under _Authentication type_.  Click **Create**.

The next steps can be done in an existing ASP .NET Core web application.

- Right-click on your project, and select **Add** > **Connected service**.
- On the left side of the tab, ensure that 'Connected Services' is selected.
- Under 'Service Dependencies', look for the ellipses (**...**) to the right of 'Microsoft identity platform'.  Click the ellipses, and choose 'Connect'.
- In the 'Microsoft identity platform' dialog, ensure the correct Tenant is selected, then click **+ Create New**.
- Enter a new display name for the app, and click 'Register'.
- Click 'Next' twice.
- Click 'Finish.'

> [!TIP]
> If there is an error which occurs after clicking 'Finish,' try clicking 'Back' and clicking 'Finish' again.

You should notice the **Program.cs** and **appsettings.json** files have been modified to reflect the new changes.  You'll also notice a new **\_LoginPartial.cshtml** View in the **Views** > **Shared** directory of your project, and a new reference to this View in the **\_Layout.cshtml** page, as highlighted below.

```
	</ul>
	<partial name="_LoginPartial" />
</div>
```

### Register Your Application

For registering your application in the [Azure portal](https://ms.portal.azure.com), you'll need administrative access to your Azure Tenant.

In order to use Azure AD authentication services within your web application, you'll need to register the web application in the Azure portal.  You'll need to provide a name, a redirect URI, and a sign-out URI for your app.  You will also get a client ID and a tenant ID that will be used in the code.  [A tutorial](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/azure-active-directory/?view=aspnetcore-6.0) has been provided for this step.  Follow the link, select _"Quickstart: Add sign-in with Microsoft to an ASP.NET Core web app"_, then under **QUICKSTART**, select the applicable framework.  For this example, select _"ASP.NET Core"_.  This will describe the process for registering your web application in Azure portal.

Under 'Authentication', which can be found in the left-hand sidebar, remember to enable 'ID tokens (used for implicit and hybrid flows)'.  After enabling ID tokens:

- Navigate to the Azure Active Directory in the left hand sidebar, and select **App registrations** > _Your app_.
- Click **Manifest** at the bottom of the pane describing your app.
- Change the value of the property 'oauth2AllowIdTokenImplicitFlow' to 'true', if it's not already.
- Click **Save**.

Be sure to assign a Redirect URL for your application to use for signin and signout authentication.  For example, you can use `https://localhost:44321/` for your Redirect URL.  Azure AD needs this URL so it knows where to place the token upon signout, so your web app can pick the token back up again when you sign in.

- Navigate to the Azure Active Directory in the left hand sidebar, and select **App registrations** > _Your app_.
- Click **Authentication**.
- Under _Platform configurations_, in the section labelled _Web: Redirect URIs_, you'll need to add sign-in and sign-out redirect URIs for your web application.
- If you're using a custom domain in addition to the default domain, you'll want to add separate sign-in and sign-out redirect URIs for each.
- If you need to add URIs for local development, you can add those under _Redirect URIs_, as well; for example, you might add "https://localhost:\<port-num\>/", "https://localhost:\<port-num\>/signin-oidc", and "https://localhost:\<port-num\>/signout-callback-oidc".

You can also create user security groups to grant select groups of users access to certain protected parts of your web application.

### Install Azure AD Services

Install the Microsoft.Identity.Web package from NuGet.  This can be done in Visual Studio 2022 by following these steps:

Open the **Tools** menu, highlight **NuGet Package Manager**, select **Package Manager Console** (PMC).

In the PMC, run the following command:
```
Install-Package Microsoft.Identity.Web
```
This package contains the middleware and services that enable Azure AD authentication in ASP.NET Core apps.

> [!TIP]
> You can also use the NuGet Manager to install packages.  Go to **Project** > **Manage NuGet Packages...**.  Click "Browse" tab at the top, and then search for a package to install.  Click on the package to install, then click the "Install" button to the right.

Be sure to also have the following packages installed:

- Microsoft.Identity.Web.UI
- Microsoft.AspNetCore.Authentication.OpenIdConnect

### Authentication Implementation

#### Modify Configuration Files

Several project files will need to be modified to utilize Azure AD authentication.

In the appsettings.json file, add a section called AzureAd with the following settings:

```
"AzureAd": {
	"Instance": "https://login.microsoftonline.com/",
	"Domain": "[Enter the domain of your tenant, e.g. contoso.onmicrosoft.com]",
	"TenantId": "[Enter the Tenant Id (Obtained from the Azure Portal. Select 'Endpoints' from the 'App registrations' blade and use the GUID in any of the URLs)]",
	"ClientId": "[Enter the Client Id (Application ID obtained from the Azure portal)]",
	"CallbackPath": "[Enter the Redirect URI (in the format of '/signin-oidc')]",
	"SignedOutCallbackPath": "[Enter the Sign out URI (in the format of '/signout-callback-oidc')]"
}
```

If you're using any security groups for your web application through Azure AD, you can add those settings, too:

```
"AzureSecurityGroups": {
	"GroupNameA": {
		"ObjectID": "[Enter the Security Group Object Id (Obtained from the Azure Portal.)]"
	},
	"GroupNameB": {
		"ObjectID": "[Enter the Security Group Object Id (Obtained from the Azure Portal.)]"
	}
}
```

Security groups will be covered in more detail in a later section.

> [!WARNING]
> It would be best not to keep these Ids stored within the appsettings.json file itself, and instead use Environment variables, a configuration file, or Azure Vault.  Details will be provided in a later section.
> Certainly, if you're using version control, it would be best never to keep these values in the appsettings.json file, even during these initial setup stages, and during development.  Instead, consider storing it in a separate secret file outside of the scope of your local repository.  This method is covered in the **Separating Sensitive Information from App Settings** section covered later on in this documentation.

Note that the Azure ClientId (also known as the AppId), TenantId, and GroupId are in the form of:  **"XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"**

Don't forget, if your web application is connecting to an external database, to include the connection string in your appsettings.

In the Startup.cs file, add the following code in the ConfigureServices method:

> [!TIP]
> If your project does not contain a Startup.cs file, you can write your configuration settings in the Program.cs file, instead.

```
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
	.AddMicrosoftIdentityWebApp(builder.Configuration.GetSection("AzureAd") ?? throw new InvalidOperationException("Azure authentication options could not be found."))
	.EnableTokenAcquisitionToCallDownstreamApi()
	.AddInMemoryTokenCaches();

builder.Services.AddControllersWithViews(options => 
{
	var policy = new AuthorizationPolicyBuilder()
		.RequireAuthenticatedUser()
		.Build();
	options.Filters.Add(new AuthorizeFilter(policy));
});
builder.Services.AddRazorPages()
	.AddMicrosoftIdentityUI();
```

> [!NOTE]
> When adding authentication to the Program.cs file, appending `.AddInMemoryTokenCaches()` can be used to store and quickly retrieve tokens without having to acquire them again, which can improve the performance of the application.
> The downside to this method is that if there are a lot of users using the application, the memory usage for caching tokens can become very high.  So, it is advisable to use in-memory token caches only for applications for which there is expected to be a low number of authorized users using it.
> `.EnableTokenAcquisitionToCallDownstreamApi()` is used if your application retrieves data from a database or another API.  Note here that while interacting with the database through a web API, the database counts as a "downstream API."  However, if you are interacting with the database directly through a database connection (for example, Entity Framework in .NET), it's just referred to as a data source, rather than an API.

You can also choose to register database context here:

```
var connectionString = builder.Configuration.GetConnectionString("ConnectionStringName") ?? throw new InvalidOperationException("Db connection string could not be found.");
builder.Services.AddDbContext<DbContextClass>(options => 
	options.UseSqlServer(connectionString));
```

_ConnectionStringName_ is the name of your connection string found in appsettings.json, or your configuration file.  Likewise, _DbContextClass_ is the name of your context class, most likely found within the **Data** folder of your project.

Be sure that your database context class extends the **DbContext** class, _not_ the **IdentityDbContext** class.  You will need to install the `Microsoft.EntityFrameworkCore` package to use **DbContext**.

> [!NOTE]
> Alternatively, you can install the following packages:
>
> - Microsoft.EntityFrameworkCore.Design
> - Microsoft.EntityFrameworkCore.Sqlite
> - Microsoft.EntityFrameworkCore.SqlServer
> - Microsoft.EntityFrameworkCore.Tools
>
> These should provide the necessary functionality for your database connections.

In the Startup.cs file, add the following code in the Configure method, after `app.UseRouting();`:

```
app.UseAuthentication();
app.UseAuthorization();
```

Also ensure the following lines exist in your Program.cs file, if you're writing a .NET Core application with Razor pages:

```
app.MapControllerRoute(
	name: "default",
	pattern: "{controller=Home}/{action=Index}/{id?}");
app.MapRazorPages();

app.Run();
```

This is the routing format used by the application.  `controller=Home` and `action=Index` provide a default Controller and Action, if they are not provided in a URL.

### Client Secret Keys

Azure Client Secrets and Certificates provide an extra layer of security to a web application utilizing Azure AD authentication.  Secrets are easier to set up, maintain, and use, but must be managed carefully and rotated regularly to ensure they remain secure.  Certificates, on the other hand, provide a higher level of security, at the cost of increased complexity in terms of setup, management, and use, but are easier to handle and rotate.  For the purposes of this documentation, we will be using Client Secrets.  Additional information on the setup, maintenance, and use of certificates may be added to another section at a later date.

Within the appsettings.json file, inside the **AzureAd** object, add a property called "ClientSecret", as seen below.

```
"AzureAd": {
	...
	"ClientSecret": "[Enter the Secret Value (the value obtained from the Client Secret in Azure Portal. Can only be seen and copied once per Client Secret.)]"
}
```

> [!WARNING]
> It would be best not to keep the Client Secret stored within the appsettings.json file itself, and instead use Environment variables, a configuration file, or Azure Vault.  Details will be provided in a later section.
> Remember, Secrets data stores present a weak point to intruders.  Proper handling and management of application secrets is highly recommended to mitigate security breaches.

In order for the application to use the Client Secret, you actually need to have a Client Secret, and have its Secret Value.  To create a Secret for your application:

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator.
- Search for and select **Azure Active Directory** or **Microsoft Entra ID**.
- Select **App registrations** in the left-hand sidebar.
- Search or browse for _your Application_ in **App registrations**, and select it. (You may need to check the **All applications** tab)
- In the left-hand sidebar, select **Certificates & secrets**.
- Click on the **Client secrets (_num_)** tab
- Click on **+ New client secret**
- Enter a Description and Expiration date for the new Client Secret, then click "Add".
- Immediately copy the value of the **Secret Value**.  This is the information that your web application will use to communicate with AzureAD.

> [!CAUTION]
> Copy and store the Secret Value to a secure location immediately.  Once a Secret is created and you have navigated away from the **Certificates & Secrets** page, the value is no longer available to be seen or copied.
> Failure to copy the Secret Value to a secure location will mean that it becomes inaccessible.  If you still need a Secret for your web application, you can make a new Secret, but it is advisable to delete the old one(s).  Any applications that use the old Secret Value will need to be updated to the new Secret.

For more details on creating a Secret for use with your web application, see [Microsoft's tutorial](https://learn.microsoft.com/en-us/azure/industry/training-services/microsoft-community-training/frequently-asked-questions/generate-new-clientsecret-link-to-key-vault) on generating a new application Secret.

### Authorization Implementation

Now that authentication is set up, authorization should be implemented next.  This is so that users may only utilize features of the web application for which they have the right permissions.

Add the following line to your Program.cs file:

```
builder.Services.AddAuthorization(options => 
{
	options.FallbackPolicy = new AuthorizationPolicyBuilder()
	.RequireAuthenticatedUser()
	.Build();
});
```

This global authorization filter provides a fallback authorization policy that requires all users to be authenticated, in the event that no other authorization policy is specified for any given resource in the application.

In order to add general authorization for authenticated users to access certain sectors of your web application, use the `[Authorize]` tag on API functions, to ensure that a user must be authenticated in order to access the resource.

> [!IMPORTANT]
> It is generally advisable to protect all API resources within your web application with a general `[Authorize]` tag, as well as setting up a global authorization filter.  This policy then requires Azure AD authentication across the entirety of your web application.

### User Membership in AD Security Groups

#### Creating Security Groups in Azure

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator.
- Search for and select **Azure Active Directory** or **Microsoft Entra ID**.
- Select **Groups** in the left-hand sidebar.
- You will need to create a new group by clicking **New group**.
- Select **Security** as the group type.
- Enter the **Name** and **Description** of the group and switch on Azure AD roles if needed.
- Select the **owners** and **members** of the group, then click **Create**.

If you have not done so already, you'll need to enable security group claims in issued authentication tokens.  To do so, follow these steps:

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator.
- Search for and select **Azure Active Directory** or **Microsoft Entra ID**.
- Select your _App configuration_.
- Click on **Manifest** in the left-hand sidebar.
- Search for `"groupMembershipClaims": null,`
- Change its value from null to `"SecurityGroup"`.
- Click **Save**.

Once you have created a new security group and assigned user members to that group, it is time to assign it to your web application.  Simply follow these steps:

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator.
- Browse to **Identity** > **Applications** > **Enterprise applications** > **All applications**.
- Search for the existing web application, and select it. (You'll want to select your application's _Service Principal_.  You can also search for this directly.)
- Select **Users and groups** in the left-hand sidebar, and then select **Add user/group** near the top.
- In the **Add assignment** pane, select **None selected** under **Users and groups**.
- Search for and select the security group you wish to assign to the application.
- Click **Select**.
- In the **Add assignment** pane, click **Assign** at the bottom of the pane.

The security user group should now be assigned to the web application.

#### Implementing Role-Based Access Control (RBAC) within your Application

Once you have your Security Group created, it is time to configure your web application to implement your Azure AD Security Group in an Authorization Policy.  To do so, you'll first need your Security Group Object ID.  If you do not have this already, you can find it by following these steps:

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator.
- Search for and select **Azure Active Directory** or **Microsoft Entra ID**.
- Select **Groups** in the left-hand sidebar.
- Search for your security group.  The Object Id should show up among the search results.  If it doesn't, click on your security group.
- Under **Object Id**, you'll find the security group's Object Id.  Copy this value.

In your appsettings.json file, you'll need to add a new object called `"AzureSecurityGroups"`, if it does not exist already, and add your Security Group Object ID to it.  See the code snippet below for an example.

```
"AzureAd": {
	...
},
"AzureSecurityGroups": {
	"GroupName": {
		"ObjectID": "[Enter the Security Group Object Id (Obtained from the Azure Portal.)]"
	}
},
"ConnectionStrings": {
	...
}, ...
```

Once you've done that, you'll need to register your new security group within the Authorization Service Collection.  Navigate to your Program.cs file within your web application, and add the following:

```
string securityGroupId = builder.Configuration["AzureSecurityGroups:GroupName:ObjectID"];

builder.Services.AddAuthorization(options => 
{
	options.FallbackPolicy = new AuthorizationPolicyBuilder()
	.RequireAuthenticatedUser()
	.Build();
	options.AddPolicy("SecurityPolicy", policy => 
		policy.RequireClaim("groups", securityGroupId));
});
```

> [!NOTE]
> When adding a new Security Group policy, `"SecurityPolicy"` can be replaced with a more meaningful policy name.  Just be sure that it matches the Authorization policy tags used throughout your application.

If you wish to add multiple Azure Security Groups to a single policy, that can be done like so:

```
string groupA = builder.Configuration["AzureSecurityGroups:GroupNameA:ObjectID"];
string groupB = builder.Configuration["AzureSecurityGroups:GroupNameB:ObjectID"];

builder.Services.AddAuthorization(options => 
{
	options.FallbackPolicy = new AuthorizationPolicyBuilder()
	.RequireAuthenticatedUser()
	.Build();
	options.AddPolicy("SecurityPolicy", policy => 
		policy.RequireAssertion(context => context.User.HasClaim("groups", groupA) || context.User.HasClaim("groups", groupB)));
});
```

You can also add multiple security policies this way, if you wish.  For example:

```
string groupA = builder.Configuration["AzureSecurityGroups:GroupNameA:ObjectID"];
string groupB = builder.Configuration["AzureSecurityGroups:GroupNameB:ObjectID"];

builder.Services.AddAuthorization(options => 
{
	options.FallbackPolicy = new AuthorizationPolicyBuilder()
	.RequireAuthenticatedUser()
	.Build();
	options.AddPolicy("SecurityPolicy", policy => 
		policy.RequireAssertion(context => context.User.HasClaim("groups", groupA) || context.User.HasClaim("groups", groupB)));
	options.AddPolicy("AdminSecurityPolicy", policy => 
		policy.RequireClaim("groups", groupB));
});
```

Next, you'll want to set up authorization on certain resources, so that only users with membership in this security group may access those resources.  This can be done by placing an `Authorize` tag above any API endpoints in your web application that only members of the security group should have access to, and assigning the security policy that was registered in your Program.cs file.  For example, 

```
[Authorize(Policy = "SecurityPolicy")]
public IActionResult Index()
{
	...
```

This attribute can also be declared on a Controller class within the MVC framework, to apply the security policy to every member of that Controller.  Example:

```
[Authorize(Policy = "SecurityPolicy")]
public class HomeController : Controller
{
	...
```

When declaring authorization policies on both a controller, and a function within that controller, a user must belong to both security policies in order to be able to access that function.  If either authorization policy check fails, the user will not be able to access that resource, and will trigger a 403 HTTP error response.

This should prevent access to any users who do not have user membership registered in that security group within the Azure portal, while granting access to those who do.  If you find that you cannot access an API endpoint with the security policy in place, and you have double-checked and confirmed that the user profile you've signed in to the web application with is authenticated within the application, and belongs to the security group in Azure, check the **Troubleshooting** section below for more information.

### Configuring an External Database Alongside Azure AD Authentication

> [!NOTE]
> This section will provide a very brief and surface level explanation for setting up database connection within the web application, as this is not entirely within the scope of this document, which focuses primarily on Azure Active Directory authentication and support.

To register an external database for use within the web application, the first thing you'll need is the connection string.  This should be detailed in the appsettings.json file.  For example, 

```
...
"AzureSecurityGroups": {
	"GroupName": {
		"ObjectID": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
	}
},
"ConnectionStrings": {
	"MyDbConn": "Server=[server_name].[domain];Database=[database_name];Trusted_Connection=true;MultipleActiveResultSets=true"
},
"Logging": {
	...
```

> [!NOTE]
> You can add `;TrustServerCertificate=true` to the end of the database connection string for debugging purposes, if you do not yet have a valid certificate set up on the database server.  However, this presents a serious security issue, and should be removed from the connection string before deploying the web application.  Work with your IT department to ensure a proper certificate is set up on the database server before the web application is ready for deployment.

Next, a database context class will need to be established within the web application.  Within an MVC framework, this will typically be found in a "Data" directory, alongside the Model, View, and Controller directories.  This can be added as a new directory if it does not already exist.  Create a new class within this directory; as an example, it can be called "MvcDatabaseContext".  See the sample below for a basic outline.

If you're using an ASP .NET Core with MVC template in the .NET Framework 6.0 within Visual Studio 2022, there is an easier and quicker way to set this up using Entity Framework.  This is covered in [a tutorial](https://learn.microsoft.com/en-us/aspnet/core/tutorials/first-mvc-app/adding-model?view=aspnetcore-6.0&tabs=visual-studio) on Microsoft's website, but the steps are outlined here, as well:

- When adding a Controller class, you can right-click on the Controller directory, and select **Add** > **New Scaffolded Item...**
- In the **Add Scaffold** dialog box, select **MVC Controller with views, using Entity Framework** and click **Add**.
- In the **Add MVC Controller with views, using Entity Framework** dialog box, select your _Model Class_ for **Model Class**, and for the **Data context class**, click the "+" sign to generate a class name.
- Click **Add** twice.

Scaffolding should automatically create the context class, in addition to the CRUD View pages for your new Controller class.

```
using AppName.Models;
using Microsoft.EntityFrameworkCore;

namespace AppName.Data
{
	public class MvcDatabaseContext : DbContext
	{
		public MvcDatabaseContext (DbContextOptions<MvcDatabaseContext> options)
			: base(options)
		{

		}

		protected override void OnModelCreating(ModelBuilder modelBuilder)
		{
			base.OnModelCreating(modelBuilder);
			modelBuilder.Entity<ModelClass>()
				.HasKey(h => h.ModelClassId);
		}

		public DbSet<ModelClass>? ModelClass { get; set; }
	}
}
```

If you have a column name in your database table which has a different name from the `ModelClass` property representing that column, you can handle that scenario with this:

```
modelBuilder.Entity<ModelClass>()
	.HasKey(h => h.ModelClassId);
modelBuilder.Entity<ModelClass>()
	.Property(p => p.ModelPropertyName)
	.HasColumnName("DbColumnName");
```

You can also add a database column as a concurrency token, for database concurrency checks, in a similar manner:

```
modelBuilder.Entity<ModelClass>()
	.HasKey(h => h.ModelClassId);
modelBuilder.Entity<ModelClass>()
	.Property(p => p.ModelPropertyName)
	.HasColumnName("DbColumnName");
modelBuilder.Entity<ModelClass>()
	.Property(c => c.RowVersion)
	.IsConcurrencyToken();
```

Don't forget to register your database context and connection string within the Program.cs file.  An example of how this is done is provided below.

```
var connectionString = builder.Configuration.GetConnectionString("MyDbConn") ?? throw new InvalidOperationException("Db connection string could not be found.");
builder.Services.AddDbContext<MvcDatabaseContext>(options => 
	options.UseSqlServer(connectionString));
```

This should provide basic information for database connection coverage.

### Managing and Rotating Secret Keys

It is important to securely handle and regularly rotate application Secrets.  Failure to do so can present a security risk and expose your application to a security breach.  This section will cover how to do this within the Azure portal and Azure Key Vault.

> [!IMPORTANT]
> [Microsoft recommends](https://learn.microsoft.com/en-us/azure/key-vault/keys/how-to-configure-key-rotation) rotating encryption keys at least once every two years to meet cryptographic best practices.  This can be done manually, or automatically via a paid service.  This section attempts to cover both practices.

#### Manual Secret Key Rotation

To rotate the web application Secret, follow these steps:

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator.
- Navigate to the **Microsoft Entra ID** service.
- Select the **App registrations** in the left-hand sidebar, then search for and select your application.
- In the left-hand sidebar, select **Certificates & secrets**, and then click **+ New client secret**.
- Provide an optional description, and then choose an expiration period (As stated above, Microsoft recommends 2 years).
- Click **Add** to create the new secret, and make note of the new **Secret Value**, as you will not be able to retrieve this value later.

Once you have done this, you must update your web application with the new secret.  If these settings are currently still stored within the appsettings.json file of the web application, you can update it with the new Secret Value as follows:

```
"AzureAd": {
	...
	"ClientSecret": "[Insert new ClientSecret Value here]"
},
```

> [!NOTE]
> Remember, the web application uses the **Secret Value**, not the **Secret ID** of the associated Secret.  This key will _not_ be in the format of "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX".

Test your application locally, or after deploying it to your production environment, to verify that your web application can authenticate successfully with the updated ClientSecret Value.

Once you have confirmed that the web application works with the new Secret, you can delete the old secret from the Azure portal, though this is optional.  To do so, simply navigate back to your application in the Azure portal, and select **Certificates & secrets** once again.  Then, identify the old client secret, and click the **Delete** button to remove it.

> [!NOTE]
> Removing an old client Secret is optional, and has benefits and drawbacks.  Some things to consider are legacy compatibility, if older versions of the web application are using the older client Secret, they will break if that client Secret is deleted.  If any configuration issues are encountered with the new Secret, keeping the old Secret allows you to revert back to a previous state, though once any configuration issues are resolved and the application is functioning as expected, the old Secret will no longer be necessary for this.  Retaining old Secret keys can also be useful for auditing purposes, if you wish to track when Secrets are rotated, and who rotated them.  However, old Secrets can provide an additional risk factor, if they are compromised.  But if a new Secret is compromised, and old Secrets are deleted, there will be no fallback option.
> Ultimately, it is up to the administrator how they would like to handle safe and secure rotation and management of client Secret keys.

#### Automated Secret Key Rotation

Once you have set up Azure Key Vault (which will be covered in the next section), or if you already have a Key Vault set up, Microsoft provides a service with an additional pay scheme to automatically handle rotating Secret Keys.  This section will highlight this method.  A [solid overview](https://learn.microsoft.com/en-us/azure/key-vault/keys/how-to-configure-key-rotation) can also be found on Microsoft's website.

Within the Azure Key Vault, you will want to set up a Key Rotation Policy for your web application's Client Secret Key.  To do this, you must:

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator.
- Navigate to your Key Vault, and select your Key, and click on **Rotation policy** to configure a key rotation policy for this key.
- Within the "Rotation policy" dialog box, you'll want to set an _Expiry time_, which is used to set the expiration date on the next, newly generated key.  Remember, Microsoft recommends at least once per 2 years.
- Check to make sure _Enable auto rotation_ is "Enabled".
- You can leave _Rotation option_ to its default setting.  See Microsoft's Key Rotation overview linked above for more information.
- Configure a _Rotation time_ for this key rotation policy.  A minimum value of 7 days is required.  This will renew your key at the given time before key expiry, as configured with _Expiry time_.
- _Notification option_ and _Notification time_ can be left to their default values.
- If you're creating a new key rotation policy on a key that already exists, you can force rotation policy manually by clicking the **Rotate now** at the top of the "Rotation policy" dialog box.

Configuring a key rotation policy can also be done at the moment of Key creation, by clicking on **Not configured** next to _Set key rotation policy_ at the bottom of the "Create a key" key creation dialog box.

For other methods of configuring automated key rotation policies, see the above provided Microsoft documentation link.

> [!NOTE]
> You can store other sensitive Key data (such as TenantID or ClientID) within your Azure Key Vault without invoking a key rotation policy.  These keys will still be encrypted at rest in Azure Key Vault, and provide security for sensitive data without having to rely on local configuration files, or appsettings.json within your web application.

A key configured with a Key Rotation policy should retain its unique URI (Uniform Resource Identifier) reference value, even after rotation, which will be covered in more detail in the **Azure Key Vault** section.

### Separating Sensitive Information from App Settings

Storing sensitive information in the appsettings.json file of the web application presents an obvious security issue.  To mitigate this major security risk, highly sensitive information can be stored within an Environment variable, configuration file, or Azure Key Vault.  For the purposes of this documentation, Azure Key Vault will be the recommended method used to store highly sensitive information.  In particular, Azure Key Vault provides a viable solution to keeping Client Secrets secure.

#### Configuration Files

A separate configuration file can be created to keep your keys separate from your source code, and removed from your git repository during development, and before establishing a Key Vault.  This documentation will cover a simple procedure for doing this within Visual Studio 2022.

In order to create a secret configuration file, open your web application project in Visual Studio, and in the _Solution Explorer_ window, right-click on your **application project**, and select **Manage User Secrets**.  This will create and open a "secrets.json" file which will be located in the `%APPDATA%\Roaming\Microsoft\UserSecrets\<user_secrets_id>\secrets.json` directory.

> [!CAUTION]
> Do note that, while the use of a User Secret does separate out sensitive information from your project and solution directories, to make it easier to avoid accidentally committing them to a git repository, this is still not a secure method of storing sensitive information.  The config file generated by Visual Studio 2022 is not encrypted (as of Nov. 2023), and if the machine on which it is stored is compromised, then the secrets are compromised, as well.
> It is advisable to use the User Secrets only during development, and as a production deployment deadline is approaching, it is advisable to switch to a more secure method of data management, such as Azure Key Vault.

Once your secret configuration file is generated, Visual Studio will open it.  If access to this config file is ever needed again, just repeat the steps to generate it in the first place, by right-clicking your **application project**, and selecting **Manage User Secrets** once again.  From here, move all the sensitive properties from your appsettings.json file over to the secrets.json file.  Once this is done, **save** both files, and close the secrets.json file.

If you used Visual Studio's built-in secret manager to generate a secret.json file, and have populated it with the sensitive data from appsettings.json, you can access this data the same way you would access it from appsettings.  As an example, in the Program.cs file, you can access the Client Secret as follows:

```
var clientSecret = builder.Configuration["AzureAd:ClientSecret"];
```

If you have utilized the _IConfiguration_ interface in your Controllers or Models using Dependency Injection, you can the sensitive data values through the configuration interface.  For example, 

```
public class ExampleModel : PageModel
{
	private readonly IConfiguration _config;

	public ExampleModel(IConfiguration config)
	{
		_config = config;
	}

	public void TestFunc()
	{
		var clientSecret = _config["AzureAd:ClientSecret"];
	}
}
```

This is, of course, just an example.  For more information on storage of sensitive information during application development, see [Microsoft's tutorial](https://learn.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-6.0&tabs=windows) on the subject.

#### Azure Key Vault

To use Azure Key Vault to secure your web application's most sensitive data, first you need to create your Key Vault, if you do not have one, already.  Be aware there are charges associated with maintaining a Key Vault; Microsoft provides a [pricing model](https://azure.microsoft.com/en-us/pricing/details/key-vault/) with information on different Key Vault storage plans.

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator.
- Select the _three horizontal lines icon_ in the upper-left corner of the portal screen, and then select **+ Create a resource**.
- Search for, and select, **Key Vault**.  Click **Add**.
- Under **Create key vault**, provide the **Subscription** and **Resource Group** the Key Vault will exist under.  If necessary, click **Create new** to create a new resource group and give it a name.
- Under _Key vault name_, enter a unique name for your key vault.  This name needs to be unique across the entire Azure platform, not just within your resource group, subscription, or even Tenant.  This name will be used to craft the custom URI which will be used to locate and access the Key Vault.
- In the **Region** drop-down, select a location.
- You can leave the other options with their default values.
- Click **Review + Create** to allow Azure to validate the data you've entered.  Then click **Create**.

Next, you'll need to populate your new Key Vault with Secret Keys.

- Navigate to the Key Vault properties page, and select **Secrets**.
- Click **Generate/Import**.
- In the **Create a secret** window, under _Upload options_, select **Manual**.  Under _Name_, enter a name for your Secret, e.g. "TenantID".
- Under _Value_, enter the Secret's value.  For example, for the "TenantID" secret, you can enter "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX".
- You can leave the other options in their default states, and click **Create**.
- Do this for each piece of sensitive data you wish to make a Secret for.

Once this is set up, the next step is to configure your Secrets in the Azure App Configuration, if you do not have one set up, already.

- In the Azure portal, search for or select **App Configuration**.
- Click **+ Create**.
- In the **Basics** tab, select the **Subscription** and create or select the **Resource group** associated with this App Config.
- Under _Location_, select the location for the app config, and then under _Resource name_, enter a globally unique name for this App Configuration.  This must be unique across the entire Azure platform, not just within your resource group, subscription, or Tenant.
- _Pricing tier_ defaults to **Free**.  You can leave this, as-is, and skip ahead to "Click **Review + create** to validate your settings,...".
- If you would like to disable public access to the **App Configuration**, consider selecting **Standard** for the _Pricing tier_, and then, in the **Networking** tab, under _Access options_, select **Disabled**.  This will allow you to create a private endpoint.
- Under **_Private Access_**, click **+ Create** to create a private endpoint.
- Click **Review + create** to validate your settings, then click **Create**.
- In the **Create private endpoint** window, select a **Subscription** and **Resource group** for your private endpoint.  Be sure these match the subscription and resource group of your Key Vault.
- Enter a **Name** for the private endpoint, then select a **Location**, **Target sub-resource**, **Virtual network**, and **Subnet** from your organization.
- Once the App Configuration is deployed, go to the App Configuration resource and select **Settings** > **Access keys**.  Make note of the primary read-only key connection string.

Once you have your App Configuration created, or if you already had an App Configuration, go to the Azure portal and select **All resources**, and then select the applicable App Configuration.

- Select **Configuration Explorer**.
- Click **+ Create**, then **Key vault reference**.
- Under _Key_, enter a reference value, e.g. "WebApp:Settings:KeyVault:\<SecretName\>".
- Leave _Label_ blank.
- Under _Subscription_, select your **Subscription**, and under _Resource group_, select your **Resource group**.
- Under _Key Vault_, select the applicable **Key Vault** with which your web application will be communicating.
- Under _Secret_, select the **Secret** for which this reference is being created.
- Click **Apply**.

Do this for each Key Vault key you plan to access with your web application.

Now, update your Program.cs file to use a Key Vault reference.  The following code should suffice, along with adding the primary read-only key connection string you took note of above to the appsettings, when setting up the App Configuration.

```
var builder = WebApplication.CreateBuilder(args);

var endpoint = builder.Configuration["ConnectionStrings:AppConfig"]

builder.Configuration.AddAzureAppConfiguration(options => 
	{
		options.Connect(endpoint)
				.ConfigureKeyVault(kv => 
				{
					kv.SetCredential(new DefaultAzureCredential());
				});
	});
```

> [!NOTE]
> You may need to install the `Microsoft.Extensions.Configuration.AzureAppConfiguration` package to avoid compiler errors with `.AddAzureAppConfiguration` in the above sample code.

Next, you can test your Azure Key Vault connection by retrieving a secret string.  You will also need to make sure you have the following packages installed:

- Azure.Identity
- Azure.Security.KeyVault.Secrets
- Azure.Core

> [!NOTE]
> `Azure.Identity` and `Azure.Core` should already be installed if you've been following along.  `Azure.Security.KeyVault.Secrets` will likely still need to be installed.  This can be done via command prompt, or by going to **Project** > **Manage NuGet Packages...** in Visual Studio, and searching for the missing packages.  Click the package, then click the _Install_ button to install that package in your web app solution.

In the Program.cs file of your web application, add the following `using` statements to the header:

```
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
using Azure.Core;
```

Add the following lines of code before the `app.MapGet` call (if it exists) in your Program.cs file, being sure to replace `<your-unique-key-vault-name>` with the `vaultUri` of your Key Vault, and `<mySecret>` with the Secret name.

```
SecretClientOptions options = new SecretClientOptions()
	{
		Retry = 
		{
			Delay = TimeSpan.FromSeconds(2),
			MaxDelay = TimeSpan.FromSeconds(16),
			MaxRetries = 5,
			Mode = RetryMode.Exponential
		}
	};
var client = new SecretClient(new Uri("https://<your-unique-key-vault-name>.vault.azure.net/"), new DefaultAzureCredential(), options);

KeyVaultSecret secret = await client.GetSecretAsync("<mySecret>");

string secretValue = secret.Value;

Console.WriteLine($"Your secret value is: {secretValue}");
```

You'll also need a reference to the App Configuration endpoint connection string in your appsettings.  For example:

```
"ConnectionStrings": {
	"DbConn": "...",
	"AppConfig": "Endpoint=[Enter the App Configuration endpoint connection string (Obtained from the Azure Portal, available in the App Configuration settings, under **Access keys**.)];Id=[App Configuration Id (Obtained from Azure Portal.)];Secret=[App Configuration secret (Obtained from Azure Portal.)]"
}
```

Your App Configuration endpoint connection string should be in the format of:  `Endpoint=https://<your-app-config-uri>.azconfig.io;Id=<your-app-config-id>;Secret=<your-app-config-secret>`.

> [!WARNING]
> The App Configuration endpoint connection string is yet another piece of sensitive information.  It would be best not to keep it stored within the appsettings, and instead, move it out to the Key Vault as soon as possible.

Once you've configured and initialized your connection to the Azure App Configuration, you can access Key Vault reference values in much the same way as you would reference appsettings configuration values.  For example, to access configuration values within a razor page:

```
@page
@using Microsoft.Extensions.Configuration
@inject IConfiguration Configuration

...

<span>@Configuration["WebApp:Settings:KeyVault:<SecretName>"]</span>
```

Or in your Controllers or Models:

```
public class ExampleModel : PageModel
{
	private readonly IConfiguration _config;

	public ExampleModel(IConfiguration config)
	{
		_config = config;
	}

	public void TestFunc()
	{
		var vaultSecret = _config["WebApp:Settings:KeyVault:<SecretName>"];
	}
}
```

> [!NOTE]
> In the configuration code sample provided above, `DefaultAzureCredential` provides access credentials that work in both local and Azure environments.  Once you've deployed your application to Azure, you'll want to use `ManagedIdentityCredential`.  For more information, see [Microsoft's tutorial on key vault references](https://learn.microsoft.com/en-us/azure/azure-app-configuration/use-key-vault-references-dotnet-core?tabs=core6x).

Microsoft also provides [a developer's resource](https://learn.microsoft.com/en-us/azure/key-vault/general/developers-guide) to utilizing Azure Key Vault with web applications, which you can peruse for more information.

#### Creating a Managed Identity

Once you've created and set up your Key Vault, and stored your security keys safely in the Vault, your web application needs a way to authenticate with the Key Vault itself, and access the Secret keys stored within.

This is where a Managed Identity comes in.  With a managed identity, your web application has a way to authenticate with the Key Vault, without exposing sensitive data and credentials.  For this documentation, a user-assigned managed identity will be set up in Azure.  To set up a managed identity, follow these steps:

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator.
- Search for and select **Managed Identities**.
- Click the **+ Create** button.
- In the "Create User Assigned Managed Identity" window, under _Subscription_ and _Resource group_, select the applicable values.
- Under _Region_, select the region for deployment of the web application.
- Under _Name_, enter a unique name for your user-assigned managed identity.
- Click **Review + create**, then click **Create**.

Once the user-assigned managed identity is created, it is time to assign it to specific Azure resources related to your web application.

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator, if you're not already.
- Search for and select **Managed Identities**.
- Select your Managed Identity.
- In the sidebar on the left-hand side, select **Azure role assignments**.
- At the top, click **+ Add role assignment (preview)**.
- In the **Add role assignment (Preview)** window, under **Scope**, select _Key Vault_.
- Under **Subscription**, select your relevant Azure subscription.
- For **Resource**, select the Key Vault created for use with this web application.
- Under **Role**, select the _Key Vault Secrets User_.  More information on Azure Roles can be found in the **Azure Key Vault Roles** section below.
- Click **Save**.
- Repeat this process for the _Key Vault Reader_ role, as well.

Your new Managed Identity should now be configured with the appropriate role assignment to access your Azure Key Vault.

> [!NOTE]
> A managed identity is managed by the Azure platform and does not require you to provision or rotate any secrets.  This means there's no overhead maintenance involved with utilizing managed identities for authenticating your web application to access secure resources.

You'll also need to ensure that your new User-Assigned Managed Identity has access to the App Config for your web application configuration.

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator, if you're not already.
- Navigate to the app configuration for your web app.
- In the left-hand sidebar, under **Settings**, select _Identity_.
- Under **User assigned**, click **+ Add**, and select your Managed Identity.
- Click **Add**.

This should ensure that your Managed Identity is configured properly to access the resources it needs in conjunction with its operation alongside your web application.

#### Configuring your Application to Utilize Managed Identity to Access the Key Vault

Azure Managed Identities can only be used in applications deployed in the Azure Portal; they do not work locally.  Once your web application is ready for deployment, it is ready to be configured with a Managed Identity to access secure data within the Azure Key Vault.

In the Program.cs file of your web application, add the following production code to configure your web application to utilize the Managed Identity to access the Key Vault and authenticate with Azure:

```
var builder = WebApplication.CreateBuilder(args);

if (builder.Environment.IsProduction())
{
	// This string variable is only necessary if using user-assigned Managed Identity; if using system-assigned Managed Identity, you don't need this, and the new ManagedIdentityCredential() declarations can be left empty.
	var identityClientId = builder.Configuration["AzureAd:ManagedIdentityClientId"];
	var endpoint = builder.Configuration["ConnectionStrings:AppConfig"];
	
	builder.Configuration.AddAzureAppConfiguration(options => 
	{
		options.Connect(new Uri(endpoint), new ManagedIdentityCredential(identityClientId))
			// Refresh the configuration settings every minute
			.ConfigureRefresh(refresh => 
			{
				// Register the Sentinel key with refreshAll set to true
				refresh.Register("WebApp:Sentinel", refreshAll: true);
				// Register Key Vault secrets which you expect to change over time
				refresh.Register("WebApp:Settings:KeyVault:GroupID_A", refreshAll: false);
				// Set the cache expiration time to one minute
				refresh.SetCacheExpiration(TimeSpan.FromMinutes(1));
			})
			// Select the configuration settings with the "Production" label
			.Select("*", "Production")
			// Resolve the Key Vault references in the configuration settings
			.ConfigureKeyVault(kv => 
			{
				kv.SetCredential(new ManagedIdentityCredential(identityClientId));
			});
	});
}
else if (builder.Environment.IsDevelopment())
{
	var endpoint = builder.Configuration["ConnectionStrings:AppConfig"];
	
	builder.Configuration.AddAzureAppConfiguration(options => 
	{
		options.Connect(endpoint)
				.ConfigureKeyVault(kv => 
				{
					kv.SetCredential(new DefaultAzureCredential());
				});
	});
}
// Staging environment left out of code sample
```

Also be sure to update your appsettings.json file with the app config endpoint (only the endpoint reference is necessary; no need to include Id or Secret):

```
"ConnectionStrings": {
	"AppConfig": "https://<your-app-config-uri>.azconfig.io"
},
...
```

And be sure to configure your appsettings.Production.json file with the Managed Identity Client ID.  This can be retrieved from your Managed Identity settings page (under **Properties**) in Azure.  Also include connection string references to your Key Vault Uri, and your database context connection.

```
"AzureAd": {
	"Instance": ...,
	"Domain": ...,
	"CallbackPath": ...,
	"ManagedIdentityClientId": "[Enter the Managed Identity Client ID (Obtained from the Azure Portal.)]"
},
"ConnectionStrings": {
	"KeyVaultUri": "https://<your-unique-key-vault-name>.vault.azure.net/",
	"MyDbConn": "@Microsoft.KeyVault(SecretUri=https://<your-unique-key-vault-name>.vault.azure.net/secrets/<your-db-connection-string-secret>)"
},
...
```

> [!NOTE]
> Don't forget to add your database connection string as a secret to the Azure Key Vault, so that you can reference it alongside other sensitive secret keys.

You'll also need to update the Program.cs file, and change the way the database connection string is registered, if you haven't already:

```
var connectionString = builder.Configuration["ConnectionStrings:MyDbConn"] ?? throw new InvalidOperationException("Connection string could not be found.");
builder.Services.AddDbContext<MvcDatabaseContext>(options => 
	options.UseSqlServer(connectionString));
```

> [!WARNING]
> Don't forget to update your appsettings.json and secrets.json files to remove the sensitive data, once the data is stored in Key Vault and your web application is set up to access those values from the vault.  Leaving that data in your web application, or even in a separate, but unencrypted file is a security hazard!
>
> You will need to update the appsettings to contain all the same references, but with the sensitive data replaced with Microsoft Azure Uri references, instead.  These are secure, as they do not reveal sensitive information, and only point to the Secrets stored within your Key Vault.  Examples of how this is done are shown below.
> 
> Do note that, out of necessity, the Managed Identity Client ID must be kept local, as it is a critical component necessary for accessing the Key Vault, and retrieving all the other keys.  It is still considered sensitive data, and should be treated as such when stored in local configuration.  However, it is the only sensitive data that should exist outside the Azure Key Vault.  The purpose of the Managed Identity Client ID is for securely accessing the Key Vault for other sensitive data, and it is the primary method in which your web application will authenticate itself with the Key Vault.

You'll next need to ensure that the Managed Identity has access permissions to the Key Vault.  You can do this by assigning a _Key Vault Secrets User_ role to the Managed Identity.  You can do this by following these steps:

- Go to the App Service in Azure.
- In the left-hand sidebar, select **Access control (IAM)**.
- Under **Role assignments**, you should see the Managed Identity with the _Key Vault Secrets User_ role.  If you do not, click on **+ Add**, then select _Add role assignment_.
- Select _Key Vault Secrets User_ role.  Click **Add**.
- Repeat this process to grant the _Key Vault Reader_ role to the Managed Identity, as well.

As one last step, don't forget to update your appsettings.Production.json file with the URIs to all the Secrets your web application will need for setup and configuration.  For example, your appsettings.Production.json file should look something like this:

```
{
	"AzureAd": {
		"Instance": "https://login.microsoftonline.com/",
		"Domain": "[Enter the domain of your tenant, e.g. contoso.onmicrosoft.com]",
		"TenantId": "@Microsoft.KeyVault(secretUri=https://<your-unique-key-vault-name>.vault.azure.net/secrets/<Tenant-Id-Secret-Name>)",
		"ClientId": "@Microsoft.KeyVault(secretUri=https://<your-unique-key-vault-name>.vault.azure.net/secrets/<Client-Id-Secret-Name>)",
		"CallbackPath": "[Enter the Redirect URI (in the format of '/signin-oidc')]",
		"ClientSecret": "@Microsoft.KeyVault(secretUri=https://<your-unique-key-vault-name>.vault.azure.net/secrets/<Client-Secret-Secret-Name>)",
		"ManagedIdentityClientId": "[Enter the Managed Identity Client ID (Obtained from the Azure Portal.)]"
	},
	"AzureSecurityGroups": {
		"GroupNameA": {
			"ObjectID": "@Microsoft.KeyVault(secretUri=https://<your-unique-key-vault-name>.vault.azure.net/secrets/<Group-Name-A-Secret-Name>)"
		},
		"GroupNameB": {
			"ObjectID": "@Microsoft.KeyVault(secretUri=https://<your-unique-key-vault-name>.vault.azure.net/secrets/<Group-Name-B-Secret-Name>)"
		}
	},
	"ConnectionStrings": {
		"KeyVaultUri": "https://<your-unique-key-vault-name>.vault.azure.net/",
		"MyDbConn": "@Microsoft.KeyVault(secretUri=https://<your-unique-key-vault-name>.vault.azure.net/secrets/<My-Db-Conn-Secret-Name>)"
	},
	"Logging": {
		...
	}
}
```

If you'd like, you can move most of these properties into the appsettings.json file, so that you can continue to develop and test locally, while still pulling the Secret values from the Key Vault and abandoning storage of Secret values and other sensitive data on your local machine.  The only attribute that would need to remain behind in appsettings.Production.json would be `ManagedIdentityClientId`.  Then, with the Program.cs code provided above, you'll be able to use the Managed Identity in Production environments, and use `DefaultAzureCredential` in the Development environment.

If you utilize this approach, you'll still need to keep the `"AppConfig"` setting locally, in a `secrets.json` file, and it will need to include the Endpoint, Id, and Secret values for your App Configuration.  See below for an example.

```
"ConnectionStrings": {
	"AppConfig": "Endpoint=[Enter the App Configuration endpoint connection string (Obtained from the Azure Portal, available in the App Configuration settings, under **Access keys**.)];Id=[App Configuration Id (Obtained from Azure Portal.)];Secret=[App Configuration secret (Obtained from Azure Portal.)]"
}
```

> [!WARNING]
> Don't store this App Configuration connection string in any file that will be committed to a repository.  This is sensitive information, and should not be exposed.

You can leave the generic AppConfig setting in appsettings.json alone.

```
"ConnectionStrings": {
	...
	"AppConfig": "https://<your-app-config-uri>.azconfig.io",
	...
```

#### Configure App Settings in Azure

Next, you'll need to configure application settings and redirect URIs in Azure.  To do so, follow these steps:

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator, if you're not already.
- Navigate to your App Service, and in the left-hand sidebar under **Settings**, click **Configuration**.
- Here, you will need to add application settings and connection strings for any settings in your appsettings.json files which contain the `@Microsoft.KeyVault(secretUri=` string.
- Click **+ New application setting**.
- In the **Add/Edit application setting** dialog, under _Name_, enter the name of the setting, exactly as it would appear in your local appsettings.json files.  E.g., if you have, for example, the following configuration:

```
"AzureSecurityGroups": {
	"GroupNameA": {
		"ObjectID": "@Microsoft.KeyVault(secretUri=..."
	}
},
...
```

- Enter this setting as "AzureSecurityGroups:GroupNameA:ObjectID" (without quotes) in the _Name_ field.
- In the _Value_ field, enter the setting value.  For example, using the above example, you would enter (without quotes):  "@Microsoft.KeyVault(secretUri=https://\<your-unique-key-vault-name\>.vault.azure.net/secrets/GroupNameA)".
- Hit **Ok**.
- Do this for each setting that uses a Uri string to reference a Key Vault Secret.  Make connection strings as Connection Strings, and not as Application settings, e.g. for a connection string, click **+ New connection string**.
- In the **Add/Edit connection string** dialog, enter the _Name_ and _Value_ for your connection string, exactly as they would appear in your application.  This means, given the following example:

```
"ConnectionStrings": {
	"MyDbConn": "@Microsoft.KeyVault(secretUri=..."
},
...
```

- You would enter the connection string _Name_ as "MyDbConn" (without quotes).
- In the _Value_ field, you would enter (without quotes): "@Microsoft.KeyVault(secretUri=https://\<your-unique-key-vault-name\>.vault.azure.net/secrets/MyDbConn)".
- Then, select the type of resource the connection string points to.
- Click **Ok**.

Next, you'll need to add redirect URIs for each path URI you have in your appsettings.  For example, suppose we have:

```
"AzureAd": {
	"CallbackPath": "/signin-oidc",
	"SignedOutCallbackPath": "/signout-callback-oidc"
},
...
```

You'll need to add this redirect URI in the Azure portal.

- Sign in to the [Azure portal] (https://ms.portal.azure.com) as an administrator, if you're not already.
- Search for "Azure Active Directory" or "Microsoft Entra ID", and select **Microsoft Entra ID**.
- In the left-hand sidebar under **Manage**, select **App registrations**.
- Search for your application.  You may need to select _All applications_ to show it.  Select your application when you find it.
- In the left-hand sidebar, under **Manage**, select **Authentication**.
- Under **Web**, and under _Redirect URIs_, add your URIs.  For this example, you'll add a URI for "https://\<your-app-config-uri\>.azurewebsites.net/" and "https://\<your-app-config-uri\>.azurewebsites.net/signin-oidc".
- Don't forget to also add your sign-out URIs.  For example, add "https://\<your-app-config-uri\>.azurewebsites.net/signout-callback-oidc".  Be sure that these sign-in and sign-out values match what you have in your appsettings.json file.
- Under _Front-channel logout URL_, be sure to enter a valid signout URI here.  This will be the default signout URI; as an example, you can change this value to "https://\<your-app-config-uri\>.azurewebsites.net/signout-callback-oidc".  **Do not use a localhost URI here!**
- Be sure to also include URIs for your custom domains.
- The signin and signout URIs need to match what you have set in your appsettings.json config file.

###### Source: [Azure Key Vault Web App Tutorial by Microsoft](https://learn.microsoft.com/en-us/azure/key-vault/general/tutorial-net-create-vault-azure-web-app).

#### Configuring your Development Environment

The last thing to do is configure your development environment to utilize the Key Vault for retrieving configuration values, so the secrets.json file can be emptied, and retired.  The following chunk of code will accomplish this:

```
else if (builder.Environment.IsDevelopment()) {
	// Establish a secret client with the Key Vault
	var keyVaultUri = builder.Configuration["ConnectionStrings:KeyVaultUri"];
	var credential = new DefaultAzureCredential();
	var client = new SecretClient(new Uri(keyVaultUri), credential);
	
	// Retrieve the secrets from the key vault and add them to the configuration
	var secrets = client.GetPropertiesOfSecretsAsync();
	await foreach (var secret in secrets) {
		var secretValue = await client.GetSecretAsync(secret.Name);
		// Iterate over all the configuration values
		foreach (var config in builder.Configuration.AsEnumerable())
		{
			// Check if the configuration value exists and contains the secret name (KeyVault Uris should contain the secret name as part of their Key Vault reference)
			if (config.Value != null && config.Value.Contains(secret.Name))
			{
				// Replace the configuration value with the secret value
				builder.Configuration[config.Key] = secretValue.Value.Value;
			}
		}
	}
}
```

After you've configured this, move the _KeyVaultUri_ configuration setting to your appsettings.Development.json file, under "ConnectionStrings".  For example:

```
"ConnectionStrings": {
	"KeyVaultUri": "https://<your-unique-key-vault-name>.vault.azure.net/"
},
```

At this point, there should be no sensitive data in your appsettings.json, appsettings.Development.json, and appsettings.Production.json files, apart from the very necessary _Managed Identity Client Id_ in your appsettings.Production.json file, as this is needed for the web application to utilize the Managed Identity.

It is now time to remove your secrets.json file, or empty it like so:

```
{}
```

At this point, your _App Config_ endpoint connection string with _Id_ and _Secret_ should also be gone from your local files, and will not be needed locally, anymore; only the _Endpoint_ itself will be necessary.

> [!TIP]
> It's advisable now to take a moment, and go through the secrets.json file, and all your appsettings files, and confirm that there are no sensitive GUIDs (in the form of "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"), ID or Password information, or other secrets.  _Managed Identity Client Id_ and _Secret URIs_ should be the only data remaining behind.

#### Azure Key Vault Roles

This section will highlight the relevant Azure access control roles where they relate to Azure Key Vault management.

- _App Configuration Data Reader_ - Grants read access to App Configuration data.
- _Key Vault Contributor_ - Grants management access to Key Vault, including Key Vault creation.  Does not grant access to keys, secrets, or certificates.
- _Key Vault Administrator_ - Grants access to managing keys, secrets, and certificates in Key Vaults.  Does not grant access to manage Key Vault resources or role assignments.
- _Key Vault Secrets Officer_ - Enables creating secrets and any management actions on certificates in the Key Vault, except managing permissions.
- _Key Vault Secrets User_ - Grants read access to secrets in the Key Vault.  The web application will need this role in order to use the secrets within the Key Vault.
- _Key Vault Reader_ - Grants read access to metadata of Key Vaults and its certificates, keys, and secrets.  It does not grant access to read sensitive values.
- _Managed Identity Operator_ - Grants access to assign managed identities to necessary resources, and allows managing managed identities.
- _Website Contributor_ - Grants access to manage websites, and publish code to the App Service associated with the application project.

For more information, see Microsoft's [Azure RBAC documentation](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide?tabs=azure-cli#azure-built-in-roles-for-key-vault-data-plane-operations).

### Deploying your Web Application to Azure

To deploy your web application to Azure, follow these steps within Visual Studio 2022:

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator, if you're not already.
- Create a new **App Service Plan**.
- Under _Subscription_, select your subscription.
- Under **App Service Plan details**, under _Name_, enter a new name for the service plan.
- Create a new **Web App**.
- Grant developer a user role, by going to the Azure portal, searching for the application, and then in the left-hand sidebar, clicking **Access control (IAM)**.
- From here, go to _Role assignments_, and assign the publishing developer the _Website Contributor_ role.
- Next, go to Visual Studio 2022.
- In the _Solution Explorer_, right-click on your project, and select **Publish**.
- In the **Publish** dialog window, select **Azure**, then click **Next**.
- If developing in Windows, select **Azure App Service (Windows)**.  Click **Next**.
- Sign in to your _Azure subscription_, and create the necessary resources.
- You'll need to select your _App Service_ from the list.  Click **Next**.
- Make sure the **Publish (generates pubxml file)** selection is selected, then click **Finish**.
- Click **Close**.
- At the top of the _App-name: Publish_ tab, click the purple **Publish** button.

Anytime you deploy the web application to Azure following this, you can just right-click on the _project name_ and select **Publish...**, then click **Publish**.  You'll also need to _Restart_, or _Stop_ and _Start_, the web application's app service in Azure.

If there are any issues with deployment, review the Application Event Logs in the Azure portal, and consult the **Troubleshooting** section down below.

### Troubleshooting

#### When running the application without debugging, a sign in page comes up, with the message "Sorry, but we're having trouble signing you in."  And the error message: "AADSTS700054: response_type 'id_token' is not enabled for the application.

This is a common error encountered when using Implicit flow and id_token is not enabled for your web application.  To resolve this issue:

- Sign in to portal.azure.com with your administrator account in your organization's Tenant.
- Navigate to the Azure Active Directory in the left hand sidebar, and select **App registrations** > _Your app_.
- Click **Manifest** at the bottom of the pane describing your app.
- Change the value of the property 'oauth2AllowIdTokenImplicitFlow' to 'true'.
- Click 'Save'.

Alternatively, you can enable the id_token under the 'Authentication' section of your app registration, by checking the box for **ID tokens** under 'Advanced Settings', and then under the 'Implicit grant and hybrid flows' section.  Don't forget to save.

#### When running the application without debugging, a sign in page comes up, with the message "Sorry, but we're having trouble signing you in."  And the error message: "AADSTS500113: No reply address is registered for the application."

A reply address, also called "Redirect URL" or "Redirect URI", needs to be registered for your web application in the Microsoft identity platform.  This is required by Azure AD, so that it knows where to post your token back so that your app can pick that token up and use it.  To resolve this issue:

- Sign in to portal.azure.com with your administrator account in your Tenant.
- Navigate to the Azure Active Directory in the left hand sidebar, and select **App registrations** > _Your app_.
- From the app's Overview page, select **Authentication**.
- Beneath 'Platform configurations', under the 'Redirect URIs' section, which can be found under 'Web', add the URL that you used in your app as the reply address.  For example, `https://localhost:44321/`.
- Click 'Save'.

Alternatively, you can use the app **Manifest**:

- Sign in to portal.azure.com with your administrator account in your Tenant.
- Navigate to the Azure Active Directory in the left hand sidebar, and select **App registrations** > _Your app_.
- Click **Manifest** at the bottom of the pane describing your app.
- Change the value of the property 'logoutUrl' to your signout callback URL; for example: `https://localhost:44321/signout-callback-oidc`.
- Change the value of the property 'replyUrlsWithType' to your signin and redirect URLs; for example:

```
"replyUrlsWithType":[
	{
		"url":"https://localhost:44321/signin-oidc",
		"type":"Web"
	},
	{
		"url":"https://localhost:44321/",
		"type":"Web"
	}
]
```

Don't forget to save your changes.

#### When running the application without debugging, a pop-up comes up that says: "Unable to connect to web server '<project name>'.  The web server is no longer running."

Click **OK**, and check the debug console.  It's likely the web application has had an issue authenticating with Azure; the console could confirm or disprove this.  For example, you could see something like `Unhandled exception. Azure.Identity.CredentialUnavailableException: DefaultAzureCredential failed to retrieve a token from the included credentials. ...` in the console.

If it's an authentication issue, check to make sure you haven't missed any steps in the setup provided in this document.  For more information, see [Microsoft's vast library of documentation](https://learn.microsoft.com/en-us/dotnet/azure/?utm_source=aspnet-start-page&utm_campaign=vside) on Azure and .NET Core.

#### When trying to access a resource with an Azure Group Authorization policy, a message is shown saying "Access Denied: You do not have access to this resource.", despite being authenticated, and belonging to the Azure Authorization Group.

Debugging code can be used to first verify whether or not the group claims are being included in the token.  Use the following example within a resource function to inspect the claims of the authenticated user, and verify the presence of the group claims:

```
var claims = User.Claims.Select(c => new { c.Type, c.Value }).ToList();
var claimsString = string.Join(", ", claims.Select(c => $"{c.Type}: {c.Value}"));

return Content($"SecurityGroup: {claimsString}");
```

If the security group claims are not present in the issued token, the group needs to be assigned to the application in the Azure portal.  This can be done by navigating to the application in the Azure portal, and following these steps:

- Click on **Manifest** in the left-hand sidebar.
- Search for `"groupMembershipClaims": null,`
- Change its value from null to `"SecurityGroup"`.
- Click **Save**.

Now, when you run your web application, be sure to sign out, then sign back in to get a new token which reflects this change.  This new setting will only take effect for new tokens issued after the change.  You may also have to wait a bit for Azure to update before cycling tokens.

#### Process has failed with the following error:  "Error, TS001: This account 'username@domain' needs re-authentication. Please go to Tools->Options->Azure Services Authentication, and re-authenticate the account you want to use.."

In Visual Studio, go to **Tools** > **Options...**.  In the **Options** dialog box, on the left-hand side, search for _Azure Service Authentication_, click the arrow to expand the node, and select _Account Selection_.  In here, make sure your account is selected, and there should be a yellow warning sign with the message:  "Re-enter your credentials".  Click the message, select your domain, and click **Continue**.  Sign in with your credentials, and then take note of the authentication code on screen, and when you receive a notification in your authenticator app on your phone, enter the code.  When you're authenticated, click **Ok** to close the **Options** window.

If nothing happens when you click **Continue** in the **Re-enter your credentials** dialog box, try closing out of Visual Studio, and rebooting it.  Then, repeat these steps to re-authenticate.

> [!TIP]
> To avoid this error, be sure to refresh your credentials every once in a while.

#### When deploying the web application to Azure, publish failed, with an error that states:  "Web deployment task failed. (Could not connect to the remote computer ("\<azure-app-domain\>") using the specified process ("Web Management Service") because the server did not respond. Make sure that the process ("Web Management Service") is started on the remote computer. Learn more at: https://go.microsoft.com/fwlink/?LinkId=221672#ERROR_COULD_NOT_CONNECT_TO_REMOTESVC.)".

Check for an error response in the app logs, Visual Studio output window, or in the error message itself.  If there's a `403 Forbidden` API error, it's likely the cause is the web application's app service is set to Private only in the Azure portal.  You'll need to set this to public, and make allowance rules for your organization's IP address to publish the web application to Azure.

If there is not a `403 Forbidden` API error, try publishing the web application again.

#### When deploying the web application to Azure, publish failed, with an error that states:  "Web deployment task failed.  (Web Deploy cannot modify the file '\<your-solution-name\>.dll' on the destination because it is locked by an external process.  In order to allow the publish operation to succeed, you may need to either restart your application to release the lock, or use the AppOffline rule handler for .Net applications on your next publish attempt.  Learn more at: http://go.microsoft.com/fwlink/?LinkId=221672#ERROR_FILE_IN_USE.  Learn more at: https://go.microsoft.com/fwlink/?LinkId=221672#ERROR_FILE_IN_USE.)"

If you run across this error when deploying the web application, 

- Sign in to the [Azure portal](https://portal.azure.com) as an administrator.
- Navigate to the App Service.
- Click **Restart**.
- Go back to Visual Studio, and publish the web application again.

#### When utilizing configuration options from the appsettings in WebApplication builder options, the configurations won't work, despite the configurations being set up properly and called correctly within the Program.cs file.

When pulling configuration options from appsettings, always set them as strings, then use them in configuration options.  Never call the configuration options directly.  For example, the following block of code will not function correctly:

```
builder.Configuration.AddAzureAppConfiguration(options => 
{
	options.Connect(builder.Configuration["ConnectionStrings:AppConfig"])
			.ConfigureKeyVault(kv => 
			{
				kv.SetCredential(new DefaultAzureCredential());
			});
});
```

The offending line that breaks is this:  `options.Connect(builder.Configuration["ConnectionStrings:AppConfig"])`.  Instead, what you should do is first set a string variable to the configuration value, then pass in the string variable itself to the configuration options, like so:

```
var endpoint = builder.Configuration["ConnectionStrings:AppConfig"];

builder.Configuration.AddAzureAppConfiguration(options => 
{
	options.Connect(endpoint)
			.ConfigureKeyVault(kv => 
			{
				kv.SetCredential(new DefaultAzureCredential());
			});
});
```

This should resolve any issues or errors relating to 'invalid connection string' parameters.

#### When deploying the web application, the call to warmup the site failed, and the Application Event Logs show an unhandled exception of type "Azure.Identity.CredentialUnavailableException" with the message "ManagedIdentityCredential authentication unavailable.  Multiple attempts failed to obtain a token from the managed identity endpoint."

If you're using a user-assigned Managed Identity, check that the resources you are trying to access with it are listed under its _Associated resources_.

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator, if you're not already.
- Search for **Managed Identities**, and select your user-assigned Managed Identity.
- In the left-hand sidebar, select **Associated resources**.
- Here, a list of the Azure resources associated with the selected Managed Identity will be shown.  Double check that your resources are listed here (you should see the AppConfig).

You can also check the App Configuration for your user-assigned Managed Identity.  To do this, follow the steps below:

- Sign in to the [Azure portal](https://ms.portal.azure.com) as an administrator, if you're not already.
- Search for your web application's **App Configuration** and select it.
- In the left-hand sidebar, select **Identity**.
- You should see your user-assigned Managed Identity listed here.
- If it is not listed, you will need to click **+ Add**.
- In the **Add user assigned managed identity** dialog, select the _subscription_ the Managed Identity is located under, and then selected the Managed Identity.
- Click **Add**.

Assuming the Managed Identity has all the appropriate roles, this should solve the issue.

#### When accessing a resource that establishes a database Context, an Internal Server Error is received: "**An unhandled exception occurred while processing the request.**  Win32 Exception: The certificate chain was issued by an authority that is not trusted.  SqlException: A connection was successfully established with the server, but then an error occurred during the login process."

This likely indicates an authentication issue with your server's Sql Certificate.  Your organization should have a valid certificate available for use; it will need to be registered with the SQL server.  The server may also need a reboot for this change to take effect.

Contact your IT department representatives for more information.

#### When running the web application locally, an exception is thrown on the `var builder = WebApplication.CreateBuilder(args);` line in the Program.cs file which states:  `System.IO.InvalidDataException: 'Failed to load configuration from file 'C:\Users\<username>\AppData\Roaming\Microsoft\UserSecrets\XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX\secrets.json'.'`

This error is caused when you empty the secrets.json file, but do not remove Visual Studio's dependency on it.  There are a few ways to handle this, and the choice is yours.

One way is to place an empty set of curly brackets in the file, e.g. `{}`, and leave it at that.

You can also disable the user secrets feature by removing `<UserSecretsId>` from your project file, and removing the `Microsoft.Extensions.Configuration.UserSecrets` package from your project.

#### Cannot sign back in after signing out.  Application Event Logs show an error that states:  `Microsoft.Identity.Client.MsalUiRequiredException: AADSTS54005: OAuth2 Authorization code was already redeemed, please retry with a new valid code or use an existing refresh token.  Trace ID: ...`

One possible solution to this problem is to have all impacted users sign out of the web application, and then re-publish the web application back to Azure again.  Sign in/sign out should be working again.

#### After updating a Key Vault Secret, the web application doesn't immediately use the updated value.

Waiting a few minutes after updating the Key Vault Secret and then restarting, or fully stopping and then starting, the web application service in Azure should have it using the new Key Vault Secret.  You may also need to set the old version(s) of the Key Vault Secret to expire, and disable them.

It is recommended to update Key Vault Secrets at the end of a work day, so the Azure application service has time to update its Key Vault references.

#### Cannot find the web tutorial for registering a web application in the Azure portal / The link has expired.

Try [this link](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-web-app-aspnet-core-sign-in), which should lead directly to the quickstart tutorial, or try searching the web for "Microsoft tutorial registering web app in Azure portal".

###### When updating this document, it is recommended that it go through review before being finalized, or pushed/merged with the main git development branch.  [This resource](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax) was utilized in the formatting of this documentation.
