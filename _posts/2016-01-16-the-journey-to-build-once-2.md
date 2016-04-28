---
title:  "The Journey to Build Once - Part 2"
date:   2016-01-16 15:41:21
categories:
- Continuous Integration
- Continuos Delivery
tags:
- DevOps
- SemVer
- Github
- Gitflow
---

#### (Preamble) Moving from SVN to Github

Not too long ago we switched from developing new projects out of our local SVN server to private Github repo. 
One of the primary reasons for the switch is that we needed to collaborate better with contractors that did not always have easy access to our network.

But the power of Github and similar hosted solutions is not just in a having a secure, generally available repository, but the extra tooling built on top.

The task of replatforming our content management system and rewriting our .com from scratch involved multiple teams of contractors. Each developer came with a different level of maturity and there was a lot of difficulty in creating a shared understanding of the work that needed to be completed, much less how it was to be architected or its implementation details.

When we decided to take the plunge to Github we opted into a variation of the [Github flow] model. Using feature branches and pull requests we were able to reach our goals on code quality while also engaging our contractors. Politically it also helped to have an honest, open dialog about what was being delivered in pull requests. 

At the risk of sounds like a spokesman or evangelist of Github I just cannot quantify how much of an impact it has had on making the project a success. The Architects, Analysts and Leads could never have accomplished so much or so quickly without great collaboration tools like this.

As the project matured we implemented feature branch builds on our CI server that allowes us greater visiblity by ensure each branch compiled and was passing unit tests.

#### [Semantic Versioning]

As I was beggining to plan out this new build once journey I realized our typical way of managing versioning *cough* manually *cough* was far from ideal. 

Under SVN we created release branches which helped provide semantic meaning, but everyone at some point forgot to update the Dev builds with the new version once the branch was cut. It was also a chore to generate release info without manually dredging the commit logs between release branches or remembering to keep a running list.

So when we started our new project we opted into the same manual model 0.1.0.{build counter}. By deploying from master we lost the habit of cutting a branch and so went away our prompt for updating the build version, it wasn't long before we saw `0.1.0.3225`

It took me longer than I would like to admit to realize that we had moved all of our rigor to the our pull requests and mentioned in passing to our Architect that it would be sweet to be able to automatically generate our SemVer based on pull request and their tags.

So for my first task in creating our Build Once pipeline was to use git history, pull request tags and the Github api to generate a semantic version to slap onto our release packages. 

These code snippets are in powershell, but it shouldn't be hard to translate them into the scripting langugage of your choice.

```powershell




```

[Semantic Versioning]:  http://semver.org/
[Github flow]:          https://guides.github.com/introduction/flow/
