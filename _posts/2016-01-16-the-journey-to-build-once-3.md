---
title:  "The Journey to Build Once - Part 3"
date:   2016-10-11 15:41:21
categories:
- Continuous Integration
- Continuos Delivery
tags:
- DevOps
- Javascript
- Configuration
- .Net
---

### Injecting javascript configuration from web.config

In a [previous post] I mentioned how we were using [envify] combined with [browserify] to set up our front end configuration values.

There are a few problems with this setup.

**It bakes in config to the js bundle**
* Making life harder when you try to debug a prod js bundle in a non-prod environment.

**It causes each environment to have its own unique hash**
* Building an environment agnostic deployment package would take longer as you have to build a seperate js bundles for each environment

One approach to solving this problem is to render the config from the server and inject it into the page with each request.

Since we already have transformations per environment set up in our project web.config it is a natural choice.

But front end developers prefer json since it is easier to parse (and read IMHO).

We can bridge the gap with a little bit of xml/json wizardry

```csharp
using Newtonsoft.Json;
using System;
using System.Xml;

namespace Application.Configuration
{
    public static class JsonConfigurationSection
    {
        // serialze an xml configuration section to json from web.config by name
        public static string GetConfiguration(string sectionName)
        {
            var document = new XmlDocument();
            document.Load(AppDomain.CurrentDomain.SetupInformation.ConfigurationFile);
            var node = document.SelectSingleNode("/configuration/" + sectionName);
            var json = JsonConvert.SerializeXmlNode(node, Newtonsoft.Json.Formatting.None, true);

            return json;
        }

        // extension add additional values not encoded in web.config
        public static string GetConfiguration(string sectionName, Func<string, string> transform)
        {
            var json = GetConfiguration(sectionName);

            if (transform != null)
            {
                json = transform(json);
            }

            return json;
        }
    }
}
```

Which allows us to change something like this:
```xml
<configSections>
  <section name="jsSettings" type="System.Configuration.IgnoreSectionHandler, System.Configuration" />
</configSections>
 <jsSettings>
    <api>
      <url>https://api.business.com/api/</url>
    </api>
    <formatting>
      <date>YYYY-MM-DD</date>
      <time>hh:mm A</time>
    </formatting>
    <insights>
      <enabled>true</enabled>
      <key>InstrumentationKeyPlaceholder</key>
      <debug>false</debug>
      <samplingPercent>100</samplingPercent>
    </insights>
  </jsSettings>
```

Into this:

```json
{
  "api": {
    "url": "https://api.business.com/api/"
  },
  "formatting": {
    "date": "YYYY-MM-DD",
    "time": "hh:mm A"
  },
  "insights": {
    "enabled": "true",
    "key": "InstrumentationKeyPlaceholder",
    "debug": "false",
    "samplingPercent": "100"
  }
}
```

By calling this:

```csharp
  JsonConfigurationSection.GetConfiguration("jsSettings");
```

Using the extension method we can also do things like injecting an AppInsight Instrumentation Key to send javascript errors and events:

```csharp
using Microsoft.ApplicationInsights.Extensibility;

namespace Application.Configuration
{
    public static class FrontEndConfiguration
    {
        private static string _value = JsonConfigurationSection
            .GetConfiguration("jsSettings", a =>
            {
                a = a.Replace("InstrumentationKey", TelemetryConfiguration.Active.InstrumentationKey);
                return a;
            });

        public static string Get()
        {
            return _value; 
        }
    }
}
```

Lastly we inject the configuration into the browser:

```cshtml
<script type="text/javascript">
    window.config = @Html.Raw(FrontEndConfiguration.Get());
</script>
```

With this setup it is easy to introduce xdt:transforms to modify our front end configuration and we can manage all the config in one spot.

This also enables us to build and hash our js bundle once and deploy to as many environments as needed.

Next on our journey we will look into transforming xml based config files for our release package.

[previous post]:  /2016/the-journey-to-build-once/
[envify]:         https://github.com/hughsk/envify
[browserify]:     http://browserify.org/
