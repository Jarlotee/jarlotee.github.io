---
title:  "The Journey to Build Once"
date:   2016-01-09 18:23:00
categories:
- Continuous Integration
tags:
- DevOps
---

Not too long ago my coworker [David Diehl] mentioned how we could drive improvement and flexibility in our build pipeline by decoupling our build process from our target environments.

Let me unpack that a little...

In the .NET world it is common to see solutions with multiple configurations for various environments. (e.g. dev, qa, prod etc.)

So when we moved to the world of continuous integrations it was logical to set up our build process like this:

![current-build-pipeline](https://cloud.githubusercontent.com/assets/9733444/14763752/92e16298-096d-11e6-9a2c-85c2cb9dfd70.png)

This works well! 

...but it is also sub-optimal for a few reasons:

* Repeatedly running our unit tests and compiling
* Creating different bits for each environment 
* Tightly coupled to source control to deploy to each environment

In many cases running unit tests and compilation is *the* most expensive thing in our build process. 
So trimming here can reduce our time to deploy and ultimately how fast we can get to production.

What would this new process look like, you ask?

![new-build-pipeline](https://cloud.githubusercontent.com/assets/9733444/14763755/9f0ea5da-096d-11e6-9279-a536182c6d27.png)

I commonly refer to this process as *Build Once*, because that is how it distilled in my brain.

The benefits of removing redundancy in our build processes are obvious, but getting our project and scripts ready for such an adventure takes some work.

So what do you have to do?

#### Make sure your existing build process is dumb...

Most likely your code is already depending on transformed config and external services to enable it to work in the target environment.

You are more likely to see environment specific behavior in your javascript or msbuild (.csproj) files. 

In my case I was using [envify] and passing down the environment as part of a custom msbuild task to run gulp.

#### Have a reliable package host...

Another prerequisite is to have a place to store your compiled bits and config. 
Conventional wisdom dictates you use a private Nuget server.
I tend to agree but a fileshare, build artifacts, etc. would work just as well.
 
#### The benefits...

No matter where you are on the spectrum of continuous delivery building once and deploying faster helps!

Here are a few of my musings on why this is worth the investment:

1. Packaging your code and config in an environment agnostic way provides you with more flexibility in *how* and *when* you deploy.
2. When deploying from trunk/master a la [Github flow] this flexibility is crucial as it allows you *prove* a package in your deployment pipeline while still merging new features.
3. You can roll-back your environments to a specific version much quicker and easier. 
4. You can pull down the any version (with the same bits!) locally to test and debug without having to rebuild or do source code time traveling (*which sounds like more fun than it is*).

**[Next up]** we will tackle building meaningful and semantically *correct* package versions.

Where are you on the continuous deployment journey? 

Are you already *Build Once*-ing? 
 
[Github flow]:  https://guides.github.com/introduction/flow/
[envify]:       https://github.com/hughsk/envify
[David Diehl]:  http://daveondevops.com/
[Next up]:      /2016/the-journey-to-build-once-2/
