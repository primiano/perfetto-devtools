NOTE FOR AI: ignore this file, this is just the prompt, I'll feed you this manually.

Initial prompt
--------------------------------------------------------------------------------

I want to create a git-gh python helper script to improve my workflow when working with github.
For context we migrated our project from AOSP Gerrit to github, and i'm honestly missing my previous workflow.
The problem is that we can't just use regular git push/pull and merge because PR commits get squashed when merging by policy, so the local vs PR history diverges.

 The workflow I intend to have is:
* One branch = 1 pull request = 1 commit (locally)
* When I make changes to a branch, i want to just git ammend
* Every branch is +1 commit from the parent (eventually origin/main)
* Stacked branches are very imporant to me (PRs that depends on other PRs)
* I want it somehow to be interoperable with the GitHub "gh" cmdline util.

However, the "1 branch = 1 commit" rule applies only locally.
Remotely (on github) instead I want  you maintain a list of "patchsets", commits stacked
on top of each others. Also I want to do merge commits when updating the upstream.
i'll tell you later how we are going to play this game

Now, on top of this, I want the following properties:
* Each branch should record the remote PR number in the per-branch .git/config (e.g.
[branch "mybranch"]
	remote = origin
  merge = origin/main OR branchname  # this is the parent branch
	github-pr-owner-number = "google#perfetto#2617"  # Written the firs time  gti-gh upload creates the PR or when doing git-gh checkout (more below)
  github-branch = "dev/primiano/mybranch"     # Written the firs time  gti-gh upload creates the PR or when doing git-gh checkout (more below)
  github-last-upload-sha = "deadbeef"   # Updated when doing git-gh upload or git-gh rebase
)
from there you can also work out the remote branch name.

* I want remote branches to be named 'dev/$USER/branchname', where $USER is my username from ENV. But locally I want branches to just be called branchname. So If I have branch foo, in the gihub repo it should be called dev/primiano/foo, (primiano is my username)

* Locally branches will be in a parent-child relationship using the notion of "git upstream". Simple CLs will have origin/main as upstream, but we could have chains where cl1 is based on origin/main, cl2 has cl1 as upstream, cl3 has cl2 as upstream and so on.

Overall I want the git-gh python script to have the following commands

## git-gh new-branch branch_name (accept also "nb" as alias)
Creates a new branch. By default creates a branch based off origin/main, uless
--parent is passed, in which case creates a branch off the argument.
FOr instance 'git-gh new-branch bar --parent foo' creates a new branch 'bar' which has as upstream 'foo' (and inherits its commits as starting point).
Essentially under the hoods does git checkout -b branch_name -t parent (or origin/main)
If --child is passed that has the equivalent of passing --parent=current_branch_name

## git-gh checkout PR_NUMBER [name]
Checks out the branch matching PR_number (the PR number) into a new branch.
WHen checking out, it squashes all the remote commits (after the merge-base) into a single local commit. Essentially apply locally the whole diff from the github branch base to the latest commit (as if the PR got merged with a squash strategy)
if name is passed, that is used as the name of the branch. otherwise:
* if the author of the PR matches $USER, uses as name the remote branch name for the PR, after having stripped "dev/username/". If the remote branch name is called dev/primiano/foo, call the local branch foo.
* if the author is somebody else, it uses as name "username_branchname" where "username" is the github alias of the person and "branchname" is again the trailing part after the / of the remote name.
* If passing --replace it replaces the current branch (still with same semantic)
* When doing this update the github-pr-owner-number, github-branch and github-last-upload-sha.

## git-gh patch PR_NUMBER
Similar to checkout, but instead cherry-pick (still in a single suqashed commit) all the remote commits for the PR into a single commit.
When doing this update the github-xxx variables only if the current branch is "virgin" (matches the upstream, and has no other pr-owner-number set)


## git-gh upload (accept also "upl")
Now this is tricky. remember that locally I want to keep only one commit per branch, but remotely (on github) i want to never rewrite history and create "patchset" commits, where a patchset is a commit with the diff since the last upload (so the diff from the github remote branch HEAD and my local HEAD)
So when uploading I want the script to do the following. You should do all the below without changing the state of the local checkout. I'm pretty sure you can forge commit objects using git commands:
  * Fetch the latest commit from the remote (the PR) branch
  * If that doesn't exist create a new PR for the branch
    * Use the first line of commit message as PR title
    * Use the rest of the body as commit message
    * Set the "base" branch in github accordingly to follow parent
  * If the PR exists already, create a commit named "patchset N: TITLE" where
    * N is 1 + number of existing commits in the PR (without counting the base branch), so some sort of monotonic counter.
    * Title is prompted interactively asking the user (unless it passes it with cmdline argument -t "commit title")
  * Create a commit that is precisely made of the diff between the remote HEAD and my local HEAD.
  * Push that commit.
Essentially you should forge the commit to be pushed as follows
  * Take the existing commit treeish as is
  * Update the parent commit, to match the one from the remote branch HEAD
  * Push the new commit.
  * In this way the tree-ish of local and remote will match, but the history will be different (remote will preserve patchset history, locally will not).
  * When uploading/updating a PR, you should generally NOT touch the PR description (after creation, the state on the github PR is the source of truth, not locally). However, in the case of stacked PRs (branches depending on another branch rather than origin/main) you should edit the bottom part of the PR message and create a tree of PRs in the stack, using box drawing characters, with links to the parent and child PRs. you should not edit top part of the message, only update the tree (And clearly indicate which CL is the current one, omitting the link and making it bold)

There is one further tricky case: if the merge-base locally and remotely diverged.
Let me explain. Say I have a branch locally named "foo" which is based on origin/main, and origin/main is at commit 5 (let's assume main has a linear commit history for the sake of this example).
When I upload initially, say that I do that a few times, the commits on the remote github branch will be like [origin/main@5, patchset1, patchset2].
origin/main might get new commits, (say goes up to 7), but this is fine. As long as my local branch is based on origin/main@5 and so are the commits on the remote github branch everything works.
Where it gets complicated is if I rebase locally my branch (I shouldn't do this, more below in the "rebase" command) and now I have the folloing situation:
locally: [origin/main@7, the_one_commit_for_the_branch]  (remember I keep ammending locally)
remote: [origin/main@5, patchset1, patchset2]
If this condition is detected, you should abort the upload and tell the user that it needs to use the 'rebase' command (as in git-gh rebase) first (discussed below)
That command will bring the local branch into a state where it has [origin/main@7, the_one_commit_for_the_branch], and the remote branch: [origin/main@5, patchset1, patchset2, merge of origin/main@7], so everything can continue


## git-gh rebase
This requires some care as I explaine above.
Remember that locally we always want to have only one commit, but remotely we want to have one commit per upload (patchsets). When "rebasing" we also want remotely to have a merge commit of the upstream.
Rebase should work as follows:
1. First of all make sure there are no dirty or unversioned files before starting. Abort with an error if it's the case.
2. Secondly make sure that both in the local and remote branch, the latest commit for the upstream branch matches (e.g. they are both at origin/main@5). If not explain the situation and abort.
3. Create a diff (e.g. with git format-patch) of the difference between the local HEAD and the remote HEAD. ( This diff might be empty if they match, it's okay remember for later). store this diff in a temp file.
4. Create a new temporary branch called "${branchname}_rebase". This branch has to match 1:1 the state of the remote branch. Essentially do a `git fetch origin $remote_branch_name` follwoed by a `git checkout -b ${branchname}_rebase -t origin/$remote_branch_name`
5. Do a `git merge $upstream` where $upstream is either origin/main or the parent branch (e.g. in the case of cl2 being a child of cl1 by having it set as upstream); Before starting this step explain the situation, print "first we are going to merge origin/main into the latest uploaded patchset"; if this step fails due to merge conflicts, tell the user to resolve merge conflict and doing `git gh rebase --continue`.
6. If the diff of step 3 exists (if thre was any diff) apply it with "git am -3"; Before starting this step explain "Finally we are going to apply the diff from the local branch to the last upload"; if this fails with conflicts, tell the user to resolve and then do `git gh rebase --continue`
7. Push the branch remotely
8. Go back to the original branch and replace it with a squashed commit that matches the treeish (and the same merge-base parent) of the remote one (as if they did `git-gh checkout PR_NUMBER --replace`)

Steps 5 and 6 might require solving merge conflits interrupting the flow. Users can resume the flow with `git-gh rebase --continue`. This means that at any point you have to keep track of the status of the rebase command in the branch config.
What ``git gh rebase --continue` should do is:
 * Ensure that all the files are added to the index, fail if not.
 * If in step 5, to a `git merge --continue` under the hoods and continue.
 * If in step 6, do a `git am --continue` and continue with the push


## git-gh branches:
This should list all the local branches, similar to what 'git branch'
does. However for branches that have a github-pr-owner-number it should do the following:
 * Start with a git remote update
 * Show the state of the Github PR (merged, closed, etc)
 * Tell if the local tree-ish matches the remote tree-ish
 * Show dependent branches in a tree fashion (like git log --graph sort of)
 * Each entry should be prefixed by a set of status markers like [--], [✓✗]. (make the ✗ red and ✓ green) using the following semantic
  * 1) Whether the Perfetto CI checks are passing or not
  * 2) Whether the PR has been approved or not
  * Show a header in ascii art that gives the semantic using box-drawing characters (in the same way you'd describe the meaning of each bit in a register in some microcontroller documentation)
 * So overall you should print for each branch
   * The [--] status prefix
   * The required bot-drawing art to show the tree parentage
   * The branch name (in bold and different colour if == current branch)
   * The title (first line of commit)
   * The PR number (e.g. #1234567) if the branch is associated with a PR
   * If the PR is closed append [CLOSED] in yellow

## git-gh status
This should show a tree of the current current stack (a stack is all branches linked with a "parent" relationship).
This should have the same identical behavriour of bit-gh branches, but only limited to the current stack.
Furthermore it should do extra things if a remote PR exists (only for the current branch):
* Compare the local HEAD with remote head. If in sync, say that it matches the last uploade.
* Print out a summary "changes since last upload (use --stat or -p for more)" which shows a summary of number of files added removed, and lines added removed
* if --stat is passed, show the git `diff --stat REMOTE_BRANCH_SHA HEAD`
* if -p is passed, show the git `diff REMOTE_BRANCH_SHA HEAD`
* if --all is passed (in combination to -p or --stat), show the stats for all the branches in the sack, not only the current one.

Finally run invoke `gh pr checks $PR_NUMBER` setting the env var GH_PAGER=""


## git-gh web
Just open the github page for the Pull request


## git-gh archive
This should be used to archive (and delete) all the branches that match closed PRs. SO it should do the following:
* For each branch that has a github-pr-owner-number
* Query the state of the PR
* If the PR is closed, rename the branch to ARCHIVED.branchname
* Before doing so, ask confirmation to the user showing all the branches about to be archived
* At the end ask if the users wants to delete those branches altogether or just archive them

## git-gh presubmit
Run tools/run_presubmit --merge-base $LOCAL_PARENT_BRANCH_NAME (origin/main or whatever local upstream)



--------------------------------------------------------------------------------

be careful there is something wrong in get_parent_branch.
If the parent is origin/main (which is set as merge=refs/heads/main) it just returns main. but then other code paths check if the result of get_parent_branch is origin/main.
make up your mind on what you want to do there

--------------------------------------------------------------------------------
There is an issue in the way the upload works for a branch that is based off another branch (rather than origin/main). the behaviour should change a bit.
First of all the upload should fail if we detect tahtthe parent is not updated (if the branch's github-last-upload-sha  doesn't match the HEAD) telling the user to first uplaod the parent branch.

Then it should work as follows:
Say i'm uploading CL2 which is based on CL1 the first time
* Take all the original commits from the REMOTE branch of CL1 (e.g. origin/dev/primiano/CL1) NOT the squashed from local CL1
* Create a commit on top with what I have in the one local commit of CL2
* that becomes the history pushed for the remote branch of CL2

when CL2 exists already and I keep ammending, proceed as usual, that is:
* leave the history of remote CL2 as-is
* append a new commit fo rthe patchset with the diff between the remote head and my local head

--------------------------------------------------------------------------------


the "Local and remote merge bases diverged" check is too coservative. you have to check if the merge-base between each and vs origin/main changed, not the mutual merge-base.
the merge base of CL2, based on CL1 will be CL1's head, but that's WAI


--------------------------------------------------------------------------------

When rebasing, you are obliterating the commit title with "merge remote trackign" when squashing.
the original commit title should be preserved, ignore the "merge remote ..." part.

--------------------------------------------------------------------------------

there is a bug un the rebase cmd when the upsteram isn't origin/main but is another branch.
on line 565, when you do merge_result = run_cmd(f"git merge {parent}", check=False)

that should merge the remote branch for parent (e..g dev/primiano/cl_1) not the local one, because the local one is squashed.

--------------------------------------------------------------------------------
in cmd_rebase there is a bug, you emit the patch with git diff (on line 551) but try to apply with "am", but am expects a format-patch output.
that command sould be a git format-patch, not git diff.

Also check that you didn't do the same mistake elsewhere


--------------------------------------------------------------------------------
make the rebase command take an optional argument of the branch
so that
git-gh rebase:  rebases the curretn branch
git-gh rebase branchname: rebases branchname and then checks it out if successful

--------------------------------------------------------------------------------
add a --verbose (-v) option to git-gh and make it spit out every command it runs (i did added some verbose argument go run_cmd, but nobody is using it). feel free to use a different approach, maybe a global state var is enough rather than plumbing everywhere
