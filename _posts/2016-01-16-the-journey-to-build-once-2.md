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

Not too long ago we switched from developing new projects out of our local SVN server to a private Github repo. 
One of the primary reasons for the switch is that we needed to collaborate better with contractors who did not always have easy access to our network.

The task of replatforming our content management system and rewriting our .COM from scratch involved multiple teams of contractors. Each developer came with a different level of maturity and there was a lot of difficulty in creating a shared understanding of the work that needed to be done, how it should be architected, and its implementation details.

When we decided to take the plunge to Github we opted into a variation of the [Github flow] model. Using feature branches and pull requests we were able to reach our goals on code quality while also engaging our contractors. Politically it also helped to have an honest, open dialog about what was being delivered in the pull requests. 

At the risk of sounding like a spokesman or evangelist of Github I just cannot quantify how much of an impact it has had on making the project a success. The Architects, Analysts, and Leads could never have accomplished so much or so quickly without great collaboration tools like this.

As the project matured we implemented feature branch builds on our CI server that allowed us greater visibility by ensuring each branch compiled and was passing unit tests.

#### Semantic Versioning using Github Pull Request Tags

As I was beginning to plan out this new [Build Once] journey I realized our typical way of managing versioning (manually) was far from ideal. 

Under SVN we created release branches which helped provide semantic meaning, but everyone at some point forgot to update the Dev builds with the new version once the branch was cut. It was also a chore to generate release info without manually dredging the commit logs between release branches or remembering to keep a running list.

So when we started our new project we opted into the same manual model `0.1.0.{build counter}`. By deploying directly from master we lost the habit of cutting a branch and lost our prompt for updating the build version. It wasn't long before we saw `0.1.0.3225` :disappointed:

I realized that we had moved all of our rigor to the pull requests. Later I mentioned to our Architect that it would be sweet to be able to automatically generate our [SemVer] based on pull requests and their tags.

I decided that this should be my first task in creating our [Build Once] pipeline. 

Now on to the codes!

These code snippets are in powershell, but it shouldn't be hard to translate them into the scripting language of your choice.

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
We start off with passing in the current commit hash and a Github token, then define a function to procure pull request data through the [Github API].

```powershell

$major = 0
$minor = 0
$patch = 0

$PullRequests = New-Object System.Collections.Stack

```

Here we set up our patch variables and declare our stack which we will use to unwind the commit history.

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

Using `match` with `git describe` allows us to utilize custom tags without interfering with the build process.

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

Here we walk the commit history since the last tag and use regular expressions to pull out the pull request numbers.

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

Lastly we work through the stack calling the Github API and looking for tags that indicate a major or minor release change.

Based on the tags present we increment the appropriate numbers and call back to the CI server (Teamcity) with the new version.

In only a few lines of code we can automatically generate a semantically meaningful version number to be used in our builds!

> Please be aware of the query limits available to you using your Github API token. In my case we already had hundreds of pull requests before we made our first git tag. Depending on how long your repository has been live it may be beneficial to start with a manually generated tag close to your current head. This will ensure you do not exhaust your token request limit on your first build.

[Build Once]:   /2016/the-journey-to-build-once/
[SemVer]:       http://semver.org/
[Github flow]:  https://guides.github.com/introduction/flow/
[Github API]:   https://developer.github.com/v3/issues/#get-a-single-issue
