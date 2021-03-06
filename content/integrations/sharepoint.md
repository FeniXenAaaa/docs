# SharePoint 2010/2013 Integration

Auth0 can help radically simplify the authentication process for SharePoint. In this tutorial, you'll learn how to add Single Sign On (SSO) to Sharepoint using Auth0. Your users will be able to log in using any of our [Social Identity Providers](@@env.BASE_URL@@/identityproviders) (Facebook, Twitter, Github, etc.), [Enterprise Providers](@@env.BASE_URL@@/identityproviders) (LDAP, Active Directory, ADFS, etc.) or with username and password.

## Setup

### 1. Adding the Integration to your account

The first thing you need to do is go to the [Third Party Apps](https://manage.auth0.com/#/externalapps/create) section in the dashboard and choose **SharePoint** from the list of apps.

![Create a new Third Party Application](//cdn.auth0.com/docs/img/sharepoint-new-app.png)

### 2. Follow the Live Documentation

On the Settings tab you'll need to enter the URL of the SharePoint Web Application and the external URL (typically the internet endpoint in your Alternate Access Mappings).

![Tutorial](//cdn.auth0.com/docs/img/sharepoint-app-tutorial.png)

The Live Documentation will ask you to execute the installation script to install our custom Claims Provider after which you'll need to enable it and complete the configuration in Central Administration:

![Central Administration](//cdn.auth0.com/docs/img/sharepoint-central-admin.png)

Note that the call to `Enable-Auth0` can be adapted to:

 - Change the unique identifier for users (eg: email or user_id)
 - Allow addition claims to be passed through to SharePoint
 - Enable or disable the default Windows Authentication

The following example also adds the Role claim to the claims mapping and allows Windows Authentication:

```
Enable-Auth0 
  -auth0Domain:"fabrikam.auth0.com" 
  -clientId:"bOFty3tWgpijnwMcltysNFqHgO1ziz1I" 
  -webAppUrl:"http://fabrikam-sp/" 
  -identifierClaimType:"http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress" 
  -claims:@(
    "Email|http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress", 
    "Role|http://schemas.microsoft.com/ws/2008/06/identity/claims/role", "Client ID|http://schemas.auth0.com/clientID", 
    "Given Name|http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname", 
    "Surname|http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname", "Picture|http://schemas.auth0.com/picture") 
  -allowWindowsAuth 
  -Verbose
```

### 3. You've nailed it.

You have configured SharePoint to use Auth0 as the SSO Broker. When your users visit your site they'll be presented with a login page showing all the connections enabled for that application.

![SharePoint Login Page](//cdn.auth0.com/docs/img/sharepoint-login-page.png)

Depending on which claims have been mapped when installing the claims provider this additional information will also be available in the user's personal settings page:

![SharePoint User Info](//cdn.auth0.com/docs/img/sharepoint-user-info.png)

## Authorization

The claims being passed through from Auth0 can also be used for authorization in SharePoint. For example, a user with the Role claim containing **Fabrikam HR** should have access or be a Contributor on a specific site.

Let's take Azure AD as an example. In this Cloud Directory users can be part of groups and David is part of Fabrikam HR.

![Azure AD Group Member](//cdn.auth0.com/docs/img/sharepoint-group-member.png)

When David logs in using his Azure AD account (and the Security Groups attribute is enabled for that connection) the group memberships will be stored in the `groups` attribute of the user's profile.

![User Groups](//cdn.auth0.com/docs/img/sharepoint-profile-groups.png)

If we want to make these groups available as Roles in SharePoint we'll need to write a [Rule](https://manage.auth0.com/#/rules) that adds this to the SAML configuration. This rule will only run for the application named **Fabrikam Intranet (SharePoint)**.

```
function (user, context, callback) {
  if (context.clientName === 'Fabrikam Intranet (SharePoint)') {
    context.samlConfiguration.mappings = {
        'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier': 'user_id',
        'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress': 'email',
        'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name': 'name',
        'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname': 'given_name',
        'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname': 'family_name',
        'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn': 'upn',
        'http://schemas.microsoft.com/ws/2008/06/identity/claims/role': 'groups'
    };
  }
  
  callback(null, user, context);
} 
```

This will add an additional outgoing claim `http://schemas.microsoft.com/ws/2008/06/identity/claims/role` containing the `groups` and which will be used by SharePoint for authorization.

When installing the Claims Provider we need to allow the Role claim to be passed through to SharePoint, by adding it to the claims mapping list:

```
Enable-Auth0 
  -auth0Domain:"fabrikam.auth0.com" 
  ...
  -claims:@(
    "Email|http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress", 
    "Role|http://schemas.microsoft.com/ws/2008/06/identity/claims/role", 
    ...) 
  ...
  -Verbose
```

By default a user won't have access to the site:

![Access Denied](//cdn.auth0.com/docs/img/sharepoint-no-access.png)

Now instead of adding that specific user to a SharePoint Group (eg: Contributors) we can now add a **Role** to a SharePoint Group. Here's a sample PowerShell script that shows how to add "Fabrikam HR" members to the Contributors group:

```
$webName = "http://fabrikam-sp"
$groupName = "Contributors"
$roleClaim = "Fabrikam HR"

$sts = Get-SPTrustedIdentityTokenIssuer "Auth0"
$claimPrincipal = New-SPClaimsPrincipal -ClaimValue $roleClaim -ClaimType "http://schemas.microsoft.com/ws/2008/06/identity/claims/role" -TrustedIdentityTokenIssuer $sts

$web = Get-SPWeb $webName
$user = New-SPUser -UserAlias $claimPrincipal.ToEncodedString() -Web $web

$group = $web.SiteGroups[$groupName]
$group.AddUser($user)
```

After adding this claim value to the Contributors group David will be able to access the site and edit its contents.

## Troubleshooting

Errors and warnings are logged to SharePoint's Unified Logging Service and tools like the [ULS Viewer](http://www.microsoft.com/en-us/download/details.aspx?id=44020) can be used to troubleshoot any issues you might have when using the Claims Provider.

![ULS](//cdn.auth0.com/docs/img/sharepoint-uls-logs.png)

And when working with additional claims and authorization it can always be useful to view the claims for the current user. [Liam Clearly](https://www.helloitsliam.com)'s [Claims Viewer Web Part](https://sharepointobservations.wordpress.com/2013/08/21/sharepoint-2013-and-adfs-2-0-test-with-claims-viewer-web-part/) can be used to troubleshoot any issues with the user's claims:

![Claims Webpart](//cdn.auth0.com/docs/img/sharepoint-claims-webpart.png)

