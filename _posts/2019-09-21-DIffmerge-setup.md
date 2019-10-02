---
title: Efficient Git Merge - Diffmerge
tags: [git, beginners, essentials, Diffmerge]
style: 
color: 
description: Diffmerge Setup guide for windows
---

Imagine your team and you have been working on loads of code and it's time to bring it all together. This is one of those dreaded (read "group collaboration") times when the developers and the leads typically sit together and start merging code, fingers crossed hoping to get it right in as little iterations as possible.

DiffMerge is a free tool used to resolve merges on a repository. It helps you do a side-by-side comparison of each file that has diffs. For example say you have a repository at a commit A (on branch master). You create a new branch of commit A and make some changes to it. Then you push it as commit B to a new-branch. While you're working on the new-branch on commit B, somebody updated the same files in the master and got it to a commit C. When you try to merge your changes upstream into the master branch, if both commit C and your commit have made changes to the same files, a merge conflict will arise. This conflict can be resolved using this diffMerge tool. The tool is freeware and allows for easy 2-way and 3-way merges.

***Full disclosure***, I've tested this on windows 7 and 10 only. AT the time of this writing in late 2019, the Diffmerge is at version 4.2.0.697.stable.

# Setup

## Download & Install Diffmerge

1. Navigate to [Diffmerge Download](https://sourcegear.com/diffmerge/downloads.php) and download diffmerge for your platform. 
2. Run the setup and install the software into your local machine. I wasn't able to change the directory it installs into because the Browse button was greyed out, but it's a relatively lightweight app so it won't be a space guzzler on your primary drive either. 
3. To double check the installation, you can open a Git Bash, cmd or a PowerShell Session and type the command `sgdm`. This should open an empty DiffMerge window.

## Setting up with git

Now that you've got Diffmerge installed and working, using some of the pre-defined git properties, we can setup Diffmerge to be the default diff and merge tool.

​​```mermaid
graph LR
A[Hard edge] -->B(Round edge)
    B --> C{Decision}
    C -->|One| D[Result one]
    C -->|Two| E[Result two]
​```
