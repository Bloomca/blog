---
layout: post
title: Git Beyond the Basics
keywords: git, git workflow, git branches, git rebase, git interactive rebase, git cherry-pick, git history, git history rewrite, software development, programming
---

I am not a git professional by any means, and my usage of git is for sure pretty primitive compared to many other experienced engineers. I would like to explain one step forward from the basics here -- how to rebase, how to cherry-pick commits from another branch and how to clean your pull request. Also, there are likely some mistakes (or at least non-precise definitions), feel free to contact me at <a href="mailto:seva.zaikov@gmail.com">seva.zaikov@gmail.com</a>!

## Basics

I'll briefly overview git itself and how it works. So, git is a CVS, which holds all information inside blobs, and all commits are just differences between previous one. It means that in order to revert one commit back, we just need to remove all changes stored in the last commit. Same works for any number of commits, even up to the very first commit -- we can easily move back and forth by commit hashes.

Git supports branches, and it is very easy to use them, which is a big help for distributed development, when a lot of people work on independent branches, and sync only in the end, fixing conflicts in files, where same lines were changed. Branch itself is just  
a pointer to a commit's hash, so we can move them without any problems. It also means that when you remove the branch, it does not really mean that -- we removed the pointer, but commit still exists, and knowing the hash, we can set the same pointer back (or another pointer, for the new branch). Tags are also just pointers, and the single difference with branches is that they are not fluid -- we tag certain commit, and it sticks with it, and branch pointer automatically moves forward after we add a new commit.

Because branches are just pointers to commits, it means that you can merge not with another branch, rather with a last commit from this branch, and it will be exactly the same thing!

- [excellent lecture about git's internals]()

## Merging

I'd explain here only the situation when you have your branch with new commits, and you want to merge it back to the master. So, if everything is alright and you have no conflicts with master, when you can just switch to the master and merge your original branch:

```sh
git checkout master
git merge feature/my-awesome-feature
```

After that git will offer you to create a new commit with default merging message. 

?? You can skip this commit::: -- Because no conflicts were introduced, we can just merge without

But if you have a conflict, git will tell you should resolve all of them, stage all changed files, and commit again. Interesting that you can do it using both approaches -- you can merge master to your branch and resolve all conflicts there, or do it in the master. Since master is usually a protected branch, so you can't push there directly, and all changes are made using mechanism of [Pull Requests]() on the service like Github or Bitbucket, the first approach is the most common. These services actually prohibit you to merge to the master, until you resolve all conflicts!

Git stores all commits and branches (which are just pointers to commits) in a form of a tree, so when we want to merge something, git starts to go back in the history of both branches, trying to find common ancestor. As soon as he finds one, he puts all new commits from both branches together and starts to apply them. We might encounter a conflict here, if new commits in both source and target branches changed same lines -- git by default has no idea how he should deal with such situations, and it is only us who have to decide. Because there was a conflict, we can't skip the commit -- we need to apply new changes, and we need a new blob for them, and new hash for this blob, which is an id for this commit.

- [Merging in git docs]()

## Rebasing

So far so good, that was pretty much basic stuff -- I think this is how majority of git users use it. So, typical workflow is something like:

```sh
git checkout -b feature/my-new-feature
git add .... # stage your changed
git commit # add your new commits
git merge origin/master # merge with remote master
git push # push all changed, and merge it after code review and CI
```

However, using this strategy we can end up with some commits which we don't necessarily want. For instance, we might want some functionality from the 3rd branch, and we merged it several times with our branch. It was merged to the master before ours, so our branch will merge to the master just fine, but these commits don't really give a lot of helpful information. Another type of commits which are common but not so helpful -- different variations of "fix", "fix after code review", "fix failing tests", where we apply some small changes which we forgot to fix our tests, linting or some improvements after code review. We can have several rounds of code review, depending on the size, and while we can just [amend new changes]() into one commit of such type, but it might harden the review (people won't be able to see only the last commit), and we still will have one more commit (not mentioning that granularity of commits for the branch and for the master might be different).

Both Github and Bitbucket offer you a "squash" button, which squashes all your commits into just one merge commit and puts all other commit messages into [additional commit message](). I personally find it as not very readable, and, moreoever, you might actually want to preserve some commits, so it will be easier in the future to search through and reason about them. 

> if you write your commit message in an editor, you can insert two newlines after your message, and type a long explanation, which will be visible in a full log. It might help in the future, because you can write all reasoning about choices there -- and unlike PRs, it will remain in the git's history forever

The answer is rewriting history, or rebasing some of your commits. It makes sense to ensure that nobody actively works on the branch when you are performing this operations -- so, ideally you rewrite the history which is only local to you, or before merging your branch. 

> Rewriting history is a dangerous thing, because you can mess other people's branches -- their new commits will have unexisting ancestors, and it is always pain in the neck. However, you can still deal with it using cherry-picking (which I'll cover in the following section), but it is an additional work anyway, so be extra careful!

One way to use rebase is to rebase target branch to your current one. Using this method, we'll put all commits which were added after we started our branch (e.g. several merged branches) in the history before our first commit, so our branch will look like we just started after all current commits in target branch and then added all our commits. It will mess up chronological order, but usually not very much, because long-term branches are not that common. In the CLI it will look like:

```sh
git rebase origin/master
```

If there is no conflicts, this will be enough. However, if any conflicts, it will look like a little bit different from merging. So, what git does internally, it gets target's pointer and starts to apply our new commits one-by-one. Each time we encounter a conflict, we have to resolve it before applying new commit. So, workflow with several conflicts will look like:

```sh
git rebase origin/master
# fix conflicts
git add ... # add all files with resolved conflicts
git commit # commit message will be prefilled in the editor
git rebase --continue
# fix conflicts
git add ... # add all files with resolved conflicts
git commit # commit message will be prefilled in the editor
git rebase --continue
# done
```

In case you mess things up and want to start from the beginning again, you can just type:

```sh
git rebase --abort
```

It will revert everything to your original branch, before you started to rebase, so you can start over (or choose other strategy).

Rebasing applies all your commits one-by-one, and it changes the history in the sense that because they have different ancestor now, commit hash is different now, so all your commits will have different hashes. Effectively it means that your branches have diverged, and now you can do two things:

- remove remote branch and push your rebased one
- push your rebased version with [force]

Removing involves too much hassle, so force push is a common approach:

```sh
git push --force origin feature/my-awesome-feature
# or alternative syntax
git push origin +feature/my-awesome-feature
```

I personally like the latter form more, but they are absolutely equivalent, so it is just a personal preference.

- [Rebasing in git docs]()
- [Force flag in git docs]()

## Interactive Rebase

Rebase has several very interesting options, but I'll focus only on one of those -- [interactive mode](). 

In the previous section we covered how to rebase target branch, so all our commits in the history will be after recently added to the target branch (like other merged branches after we started our own). But we still have no idea how to rewrite history of our branch and how to remove unwanted commits. Interactive mode helps us exactly with it.

We get into interactive rebase mode just adding `-i` flag, and we could do it in the previous chapter:

```sh
git rebase -i origin/master
```

However, interactive rebase usually is used for changing only our branch, so I will explain how to do so using this task. Let's imagine we have our branch with 6 commits, which we want to combine together before merging into master. So, in order to start interactive rebase of our branch, we need to pick the first commit, which we want to remain untouched (the last one in target branch usually). We can provide a hash, but we can just use git's references:

```sh
git rebase -i HEAD~6 # get commit which is 6 commits
# back from the last in the current branch

git rebase -i ac7xq98 # will work the same way
```

??? PICTURE WITH REBASE

As you can see, git provides you a quick description to all possible options. I'll list them here as well:

- pick (p) -- get commit as it is
- reword (r) -- get commit, but change the message
- edit (e) -- get commit, but stop there, so we can add/remove files, and reword commit message as well
- squash (s)

In the editor we need to replace `pick` with options we want to apply. You can use shorthand versions as well, with just one letter, there is no difference at all. So, let's work on our branch and change it:

??? PICTURE WITH SELECTED OPTIONS

I chose ........


```sh
git rebase -i HEAD~6 # start to rebase our branch
# chose as on the picture
git rebase --continue
# changed commit message for the first commit
# changed commit message for the third commit
# done!
```

??? PICTURE WITH GIT LOG AFTER CHANGING

We rebased master to our branch, and now new transformed 6 commits in our branch to just 3, more meaningful and not so focused on implementation details. Now we can merge it to the master and have better history in the future, so somebody who will support it can say thanks to us!

- [Interactive rebase in git docs]()
- [Fixup in git docs]()


## Cherry-Pick

Rebasing will solve 95% of our interactions with history, when we want to put other commits before ours, or we want to rewrite our commits to make more sense for the master branch, not just for development.
But sometimes we digress too much, and either the whole branch becomes irrelevant, or we just apply too much of unnecessary stuff; some code we'd like to save, though (and we know exactly which commits are interesting for us).

While can we can definitely just copy needed stuff by hand, or rewrite, there is another option which is called cherry-picking. It does exactly as it is named -- we pick one (or two, or range) commit, just by hash, and apply to our current branch. Git will get the diff stored in the blob of given commit, and try to apply it to our current one, and if we have any conflicts, we'll resolve them in a usual way.
Commit will have the same commit message, so if you want to change it, you can do interactive rebase with reword option (in case you cherry-picked several commits), or just `git commit --amend`, which will open an editor and you will be able to rewrite the message.

This is not the most common operation, so people in general are not very keen on it, but in reality there is nothing to worry about. All you need just a hash of your commit, and that is it!

```sh
git cherry-pick a523xct # put your hash commit here
```

That will do the trick, that simple! In case you have several commits, you can apply them one by one, or use [range of commits]().

How to get commit hash? You can switch to your branch, and open `git log`, which will show all your commits, and you can pick needed hashes. In case commits are pushed somewhere, and you know name of the branch, you can either fetch the branch and checkout there, or just go to the web interface and check commits in this branch.

In case your branch is only local, you can get list of all local branches using `git branch` (in case you want to see also remote branches, use `git branch --all`). And the last case, when the local branch was removed, all information can be found in [git reflog](), which contains all information about head reference (active commit). It stores all information for about 30 days, so you can find checkout to another branch, or needed commit, and it will contain a hash which you need.

As I said, this is a pretty rare operation, but sometimes it can save you a lot of time and headache, especially if changes are not trivial and located in several commits. Hope one day this short guide will help you!

- [Cherry-pick in git docs]()
- [Git reflog]()
