---
title:  "The Journey to Build Once - Part 4"
date:   2016-11-02 15:41:21
categories:
- Continuous Integration
- Continuos Delivery
tags:
- DevOps
- Configuration
- .Net
---

We need to gather all the deployable assets together in order to fulfill the Build Once requirement.

Here is what each Build Once Artifact should contain:

1) Compiled Code
2) Static Assets (`js`, `css`, `images`...)
3) All Environment Specific Configs
4) Metadata including version number (ususally in your nuget config or zip name)

Here is an example package structure:
```
App_{Version #}\
-- code\
---- { build artifacts }
-- config\
---- { environment specific config }
-- scripts\
---- { deploy scripts (as needed) }
-- metadata.xml
```

#### Transforming configuration for all environments

Visual Studio will automatically transform your `[web|app].config` file based on your current build profile (if the names match).

Our Build Once packages need to have all config transforms available so we can deploy to any environment.

To accomplish this we need to understand the msbuild language and inject a bit of code and config into our project file.

I am happy to introduce the [BuildOnce library] to simplify this task.

For those familiar with msbuild and custom tasks you can find the [source here].

Everyone else can install the [nuget package], set the variables and create your first Build Once package.

Here is what is happening:

1) Using the project ( `.csproj` ) file we identify all the tranforms ( `web.[environent].config` ) 
2) For each of these transforms find the parent ( e.g. `web.config` )
3) Run the transform
4) Create an `[environment]` specific folder and move all of the relevant config files to that folder

As an example, if your project is setup with the following structure

```
app\
-- custom.config
---- custom.prod.config 

web.config
-- web.dev.config
-- web.qa.config
-- web.prod.config
```

After executing your build with `/p:DeployOnceEnabled=True` flag you will get a folder containing the following

```
dev\
-- web.config

qa\
-- web.config

prod\
-- app\
---- custom.config
-- web.config 
```

> Note: As seen above using the BuildOnce package will handle web.config, app.config and any other custom configs you have specified.
> It will also keep track of relative paths to ensure that deployment is as easy as possible. 


Once you have packaged everything up you are ready to build once and deploy as many times as you like.

#### Final thoughts

The strategy has two over-arching benefits:
1) Saves time/cycles by eliminating redundant compilation
2) Provides flexibiliy in deployment by allowing you to build from a library of artifacts

I have observed that the majority of the effort with using this Continuous Delivery is in preparing your project.
Changes to the deployment scripts are often simplified by this process and you can use anything from nuget to a file share as a repository.

Cheers!

[source here]:  https://github.com/Jarlotee/BuildOnce
[BuildOnce library]:  https://github.com/Jarlotee/BuildOnce
[nuget package]: https://www.nuget.org/packages/BuildOnce
