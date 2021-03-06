# PayPal .NET SDK

[![Build Status](https://ci.appveyor.com/api/projects/status/github/paypal/paypal-net-sdk?svg=true)](https://ci.appveyor.com/project/paypal/paypal-net-sdk)

The PayPal .NET SDK makes it easy to add PayPal support to your .NET web application and is built on [PayPal's REST APIs](https://developer.paypal.com/docs/api/).

> **Before using this SDK, please be aware of the [existing issues and currently available or upcoming features](https://github.com/paypal/rest-api-sdk-python/wiki/Existing-Issues-and-Unavailable%5CUpcoming-features) for the PayPal REST APIs.**

## Contents

* [Upgrade Instructions](#upgrade-instructions)
* [Prerequisites](#prerequisites)
* [Getting Started](#getting-started)
  * 1. [Download the Dependencies](#1-download-the-dependencies)
  * 2. [Configure Your Application](#2-configure-your-application)
  * 3. [Make Your First Call](#3-make-your-first-call)
* [NuGet](#nuget)
* [License](#license)

## Upgrade Instructions

> **ATTENTION:** If you are upgrading from the previous REST API SDK, you will need to make the following changes to your project.  This was necessary to avoid namespace conflicts with the previous Core SDK that has since been integrated into this library.

### Web.config and App.config changes

The following section in your **web.config** or **app.config** needs to be changed from:
````xml
<section name="paypal" type="PayPal.Manager.SDKConfigHandler, PayPalCoreSDK" />
````

...to:
````xml
<section name="paypal" type="PayPal.SDKConfigHandler, PayPal" />
````

### Namespace changes

| Old Namespace | New Namespace |
| ------------- | ------------- |
| `PayPal.Manager` | `PayPal` |
| `PayPal.Exception` | `PayPal` |
| `PayPal.OpenIdConnect` | `PayPal.Api` |

### Classes that have switched namespaces

| Class | Old Namespace | New Namespace |
| ----- | ------------- | ------------- |
| `APIContext` | `PayPal` | `PayPal.Api` |
| `BaseConstants` | `PayPal` | `PayPal.Api` |

## Prerequisites

* Visual Studio 2010 or higher
* [NuGet](#nuget)

## Getting Started

### 1. Download the Dependencies

To begin using this SDK, first download this SDK from NuGet.

**Windows Command-Line**
````
NuGet Install -Package PayPal
````

**NuGet Package Manager Console**
````
Install-Package PayPal
````

Optionally, also download [log4net](https://www.nuget.org/packages/log4net/) to give your application enhanced logging capabilities.

**Windows Command-Line**
````
NuGet Install -Package log4net
````

**NuGet Package Manager Console**
````
Install-Package log4net
````

Once all the libraries are downloaded, simply add the following libraries to your project references (**Project** > **Add Reference...**):
 * PayPal.dll
 * Newtonsoft.Json.dll
 * log4net.dll

### 2. Configure Your Application

When using the SDK with your application, the SDK will attempt to look for PayPal-specific settings in your application's **app.config** or **web.config** file.

````xml
<configuration>
  <!--
  Specifies what user-defined sections will appear in this file, as well as which objects internally are able to access the sections.
  NOTE: This element MUST be the first child under the root element in the *.config file.
  -->
  <configSections>
    <section name="paypal" type="PayPal.SDKConfigHandler, PayPal" />
    <section name="log4net" type="log4net.Config.Log4NetConfigurationSectionHandler, log4net"/>
  </configSections>

  <!-- PayPal SDK settings -->
  <paypal>
    <settings>
      <add name="mode" value="sandbox"/>
      <add name="connectionTimeout" value="360000"/>
      <add name="requestRetries" value="1"/>
      <add name="clientId" value="EBWKjlELKMYqRNQ6sYvFo64FtaRLRR5BdHEESmha49TM"/>
      <add name="clientSecret" value="EO422dn3gQLgDbuwqTjzrFgFtaRLRR5BdHEESmha49TM"/>
    </settings>
  </paypal>

  <!-- log4net settings -->
  <log4net>
    <appender name="FileAppender" type="log4net.Appender.FileAppender">
      <file value="my_app.log"/>
      <appendToFile value="true"/>
      <layout type="log4net.Layout.PatternLayout">
        <conversionPattern value="%date [%thread] %-5level %logger [%property{NDC}] %message%newline"/>
      </layout>
    </appender>
    <root>
      <level value="DEBUG"/>
      <appender-ref ref="FileAppender"/>
    </root>
  </log4net>
  
  <!-- 
  App-specific settings. Here we specify which PayPal logging classes are enabled.
    PayPal.Log.Log4netLogger: Provides base log4net logging functionality
    PayPal.Log.DiagnosticsLogger: Provides more thorough logging of system diagnostic information and tracing code execution
  -->
  <appSettings>
    <!-- Diagnostics logging is only available in a Full Trust environment. -->
    <!-- <add key="PayPalLogger" value="PayPal.Log.DiagnosticsLogger, PayPal.Log.Log4netLogger"/> -->
    <add key="PayPalLogger" value="PayPal.Log.Log4netLogger"/>
  </appSettings>
  
  <system.web>
    <trust level="High" />
  </system.web>
</configuration>
````

The following are values that can be specified in the `<paypal>` section of the **app.config** or **web.config** file:

| Name | Description |
| ---- | ----------- |
| `mode` | Determines which PayPal endpoint URL will be used with your application. Possible values are `live` or `sandbox`. |
| `endpoint` | Overrides the default REST endpoint URL as well as `mode`, if set. |
| `oauth.EndPoint` | Overrides the default endpoint URL used for gettings OAuth tokens. |
| `requestRetries` | The number of times HTTP requests should be attempted by the SDK before an error is thrown. Default value is `3`. |
| `connectionTimeout` | The amount of time (in milliseconds) before an HTTP request should timeout. Default value is `30000`. |
| `clientId` | Your application's **Client ID** as specified on your PayPal account's [My REST Apps](https://developer.paypal.com/webapps/developer/applications/myapps) page for your specific application. |
| `clientSecret` | Your application's **Client Secret** as specified on your PayPal account's [My REST Apps](https://developer.paypal.com/webapps/developer/applications/myapps) page for your specific application. |
| `proxyAddress` | The address for a proxy server your application must tunnel through in order to connect with PayPal. |
| `proxyCredentials` | If `proxyAddress` is set, use this field to specify the username and password for the proxy server. The format must be `username:password`. |

### 3. Make Your First Call

**Authenticate With PayPal**

Before you can begin making various calls to PayPal's REST APIs via the SDK, you must first authenticate with PayPal using an **OAuth access token** that can be used with each call.  To do this, you will need to use the `OAuthTokenCredential` class.

````c#
using PayPal.Api;

// Get a reference to the config
var config = ConfigManager.Instance.GetProperties();

// Use OAuthTokenCredential to request an access token from PayPal
var accessToken = new OAuthTokenCredential(config).GetAccessToken();
````		
> **NOTE:** It is not mandatory to generate the `accessToken` with every call using the SDK. Typically, the access token can be generated once and reused until it expires.

For more information on the access token and how it is used in calls to PayPal, refer to [How PayPal Uses OAuth 2.0](https://developer.paypal.com/docs/integration/direct/paypal-oauth2/).

**Configure an APIContext Object**

The `APIContext` class makes it easy for developers to customize how calls are being made to PayPal.  Every PayPal resource method available with this SDK uses an `APIContext` object.

````c#
var apiContext = new APIContext(accessToken);
````

This object can be further setup to include custom configuration settings as well as include specific HTTP headers, allowing for more flexibility and customization in how HTTP requests are made using this SDK.

````c#
// Initialize the apiContext's configuration with the default configuration for this application.
apiContext.Config = ConfigManager.Instance.GetProperties();

// Define any custom configuration settings for calls that will use this object.
apiContext.Config["connectionTimeout"] = 1000; // Quick timeout for testing purposes

// Define any HTTP headers to be used in HTTP requests made with this APIContext object
if(apiContext.HTTPHeaders == null)
{
  apiContext.HTTPHeaders = new Dictionary<string, string>();
}
apiContext.HTTPHeaders["some-header-name"] = "some-value";
````

> **NOTE:** When using `APIContext.HTTPHeaders`, be aware that some headers will be overwritten depending on the SDK call (e.g. `Authorization`, `Content-Type`, and `User-Agent`).

**Use the APIContext Object to Make SDK Calls**

Now that you have your `APIContext` object setup, it's time to make your first call with the SDK.  The following is a simple example of how to get the details of a specific PayPal payment.

```c#
var payment = Payment.Get(apiContext, "PAY-0XL713371A312273YKE2GCNI");
```

To get more code samples for using the SDK with the various PayPal features, refer to the [Samples](https://github.com/paypal/PayPal-NET-SDK/tree/master/Samples) project in this repository.

For more information on what features are supported by this SDK, refer to the [REST API Reference](https://developer.paypal.com/docs/api/) page on [developer.paypal.com](https://developer.paypal.com/).

## NuGet 

[NuGet](http://www.nuget.org) is a Visual Studio extension that makes it easy to add, remove, and update libraries and tools in Visual Studio projects that use the .NET Framework.  If you develop a library or tool that you want to share with other developers, you create a NuGet package and store the package in a NuGet repository. If you want to use a library or tool that someone else has developed, you retrieve the package from the repository and install it in your Visual Studio project or solution. When you install the package, NuGet copies files to your solution and automatically makes whatever changes are needed, such as adding references and changing your app.config or web.config file. If you decide to remove the library, NuGet removes files and reverses whatever changes it made in your project so that no clutter is left.

To install and configure NuGet with your version of Visual Studio, please refer to the [NuGet Installation Guide](http://docs.nuget.org/docs/start-here/installing-nuget).

## License

* PayPal, Inc. SDK License - [LICENSE.txt](https://github.com/paypal/PayPal-NET-SDK/blob/master/LICENSE.txt)

[![Bitdeli Badge](https://d2weczhvl823v0.cloudfront.net/paypal/PayPal-NET-SDK/trend.png)](https://bitdeli.com/free "Bitdeli Badge")
