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


## Configuring Git


Now that you've got Diffmerge installed and working, using some of the pre-defined git properties, we can setup Diffmerge to be the default diff and merge tool.

>       Some of the configurations are specific to the terminal you use on windows. This tutorial configuration assumes you're using git bash. For other git distributions/ terminals, [see here](https://sourcegear.com/diffmerge/webhelp/sec__git__windows.html) .

The following commands would suffice:

```bash
# To configure diffmerge as the default "Difftool"
$ git config --global diff.tool diffmerge
$ git config --global difftool.diffmerge.cmd "C:/Program\ Files/SourceGear/Common/DiffMerge/sgdm_cygwin.sh -p1=\"\$LOCAL\" -p2=\"\$REMOTE\" --title1="Original" --title2="Modified""

# To configure diffmerge as the default "mergetool"
$ git config --global merge.tool diffmerge
$ git config --global mergetool.diffmerge.cmd "C:/Program\ Files/SourceGear/Common/DiffMerge/sgdm_cygwin.sh -merge -result=\"\$MERGED\" -p1=\"\$LOCAL\" -p2=\"\$BASE\" -p3=\"\$REMOTE\" --title1="CurrentBranch" --title2="Result" --title3="IncomingBranch""
$ git config --global mergetool.diffmerge.trustExitCode true    # Git should trust the merge exit code returned by the mergetool
$ git config --global mergetool.keepBackup false    # Do not keep the .orig backup file post merge
```

# Usage

## Background

1. You've got a respository called Test with a master branch and a few commits.

        <div class="mermaid">
        graph TB
            subgraph master
            baseCommit-->layout-->profile
            end
        </div>

2. Next thing you create a new branch to create a search feature. Make some code changes and commit them.

    ```bash
    $ git checkout -b searchBranch
    # You create the super awesome search functionality, the next best thing to google perhaps?
    $ git add .
    $ git commit -m "search 0.1"
    ```


    <div class="mermaid">
    graph TB
        subgraph master
        baseCommit-->layout-->profile
        end
        subgraph searchBranch
        baseCommit-->layout-->profile-->search1.0
        end
    </div>

3. Suddenly you're told you need to integrate search for Usernames that's gone live in the master branch!! That basically means:

    <div class="mermaid">
    graph TD
        Switch to the master branch --> Get the latest code --> Switch over to your branch --> Merge the latest code into your --> Test and BREAK NOTHING
    </div>


    <div class="mermaid">
    graph TB
        subgraph master
        baseCommit-->layout-->profile-->usernames
        end
        subgraph searchBranch
        baseCommit-->layout-->profile-->search1.0
        end
    </div>