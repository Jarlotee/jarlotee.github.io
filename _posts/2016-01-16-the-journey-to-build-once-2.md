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

#### Semantic Versioning using Github Pull Request Tags

As I was beggining to plan out this new build once journey I realized our typical way of managing versioning *cough* manually *cough* was far from ideal. 

Under SVN we created release branches which helped provide semantic meaning, but everyone at some point forgot to update the Dev builds with the new version once the branch was cut. It was also a chore to generate release info without manually dredging the commit logs between release branches or remembering to keep a running list.

So when we started our new project we opted into the same manual model 0.1.0.{build counter}. By deploying from master we lost the habit of cutting a branch and so went away our prompt for updating the build version, it wasn't long before we saw `0.1.0.3225`

It took me longer than I would like to admit to realize that we had moved all of our rigor to the our pull requests and mentioned in passing to our Architect that it would be sweet to be able to automatically generate our [SemVer] based on pull request and their tags.

So for my first task in creating our Build Once pipeline was to use git history, pull request tags and the Github api to generate a semantic version to slap onto our release packages. 

These code snippets are in powershell, but it shouldn't be hard to translate them into the scripting langugage of your choice.

```powershell
[CmdletBinding()]
Param(
    [Parameter(Mandatory=$true)]
    [string]$CurrentCommit,
    [Parameter(Mandatory=$true)]
    [string]$GitHubCredentials
)

function GetPullRequest($PullRequestNumber)
{
    $Credentials = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes($GitHubCredentials))
    
    $Headers = @{ 
        Accept = "application/vnd.github.v3+json";
        Authorization = "Basic $Credentials"
    }

    $Response = Invoke-RestMethod -Uri "https://api.github.com/repos/CarMax/Carmax.com/issues/$PullRequestNumber" -Headers $Headers 
    
    return $Response
}

```
We start off with by passing in the current commit hash and a Github token, then define a function to aquire pull request data through the Github API.

```powershell

$major = 0
$minor = 0
$patch = 0

$PullRequests = New-Object System.Collections.Stack

```

Here we set up our patch variables and decalare our stack which we will use to unwind our commit history.

```powershell

$RevisionRange = "head" 

# Get the nearest tag
$LastTag = & git describe --abbrev=0 --tags --match "[0-9]*.[0-9]*.[0-9]*"

if($LastTag)
{
    Write-Verbose "Last commit tag is $LastTag"
    $LastTagCommit = & git rev-list -n 1 $LastTag
    Write-Verbose "Tag $LastTag points to $LastTagCommit"
    $RevisionRange = "$LastTagCommit..head"
    
    #Parse LastTag into Major Minor Patch
    $Version = $LastTag.Split(".")
    $major = [int]$Version[0]
    $minor = [int]$Version[1]
    $patch = [int]$Version[2]

    Write-Verbose "The last version parsed is $major.$minor.$patch"
}

```

By using git tags to signify our releases we have a starting point with our last version and can circumvent walking the entire tree to create the next SemVer.

Using `match` with `git decribe` allows us to still utilize custom tags without interfearing with the build process.

```powershell
$CommitMessages = & git log --pretty=%s $RevisionRange 

foreach($message in $CommitMessages)
{
    if($message -match "Merge pull request #(?<pr>\d*) from.*")
    {
        Write-Verbose "Found PR $($matches.pr)"
        $PullRequests.Push($matches.pr)
    }
}

```

Here we walk the commit history since the last tag and use regular expressions to pull out the PR numbers

```powershell

while($PullRequests.Count -gt 0)
{
    $pr = $PullRequests.Pop()
    $githubPR = GetPullRequest -PullRequestNumber $pr

    if(-not (($githubPR.labels | where { $_.name -eq "major-version"}) -eq $null))
    {
        Write-Verbose "Adding major version"
        $major++
        $minor = 0
        $patch = 0
    }
    elseif(-not (($githubPR.labels | where { $_.name -eq "minor-version"}) -eq $null))
    {
        Write-Verbose "Adding minor version"
        $minor++
        $patch = 0
    }
    else
    {
        Write-Verbose "Adding patch version"
        $patch++
    }
}


Write-Host "##teamcity[buildNumber '$major.$minor.$patch']"



```

Lastly we work through the stack calling the github api and looking for tags that indicate a major or minor release change.

Based on the tags present we incriment the right numbers and in this case call back to Teamcity to let it know we have calculated the new version for the build.

So there you have it dynamically created semantic versions for your builds, a key to Build Once, but also beneficial for anyone using feature branches. 

> Please be aware of the query limits available to you using your Github API token. In my case we already had hundreds of pull requests before we made our first git tag. Depending on how long your repository has been live it may be beneficial to start with a manually generated tag close to your current head. This will ensure you do not exahust your token request limit on your first build.

[SemVer]:  http://semver.org/
[Github flow]:          https://guides.github.com/introduction/flow/
