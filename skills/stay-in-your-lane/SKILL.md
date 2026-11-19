---
name: stay-in-your-lane
description: Tells the agent that other agents are working on the same branch and working tree at the same time, so changes it did not make are expected and are to be left alone while it finishes and commits its own work. Stays in force for the rest of the session. Use when the user types /stay-in-your-lane, says "stay in your lane", or says other agents or sessions are working on this branch.
---

# Stay in your lane

Other agents are working in this same branch and working tree right now. You will see files changed that you did not change, new files you did not create, and commits you did not make. That is normal here: it is someone else's work in progress, not a problem to look into.

This holds from now until the session ends, not only for the current task. Tell the user in one line that you have it, then carry on with your work.

## Your lane

Your lane is the work this user gave this session: the files you created, and the changes you made to existing files. Everything else in the working tree belongs to someone else.

Keep track of what you touch, so that at commit time you know which files are yours without guessing. If you are already mid-session when this is invoked, work out your own files from what you have done so far in this conversation, not from `git status`, which now shows everyone's work mixed together.

## Work that is not yours

Carry on as if it were not there:

- Do not revert, fix, reformat, finish, or tidy it, even when it looks broken or half done. It probably is half done.
- Do not investigate where it came from, and do not report it to the user. They know.
- Do not let it stop you. Complete your task and commit your work regardless.

When a type check, lint run, or test fails only because of someone else's files, it is not your failure and not yours to fix. Make sure your own files are clean, go on, and mention the failure in one line of your final report.

## Committing

Commit your own work when it is done, as you normally would, with these limits:

- Stage your files by path, one by one. Never stage everything at once (`git add -A`, `git add .`, `git commit -a`), which sweeps other agents' unfinished work into your commit.
- Before committing, look at what is staged and confirm every file in it is yours.
- Leave other agents' files exactly as they are: unstaged, uncommitted, unmodified.

Never do anything that moves or rewrites work you do not own: no stash, no hard reset, no restoring or checking out files that are not yours, no cleaning untracked files, no rebase, no amending or rewriting commits, no force push, no switching branches. If a push is rejected, or the branch has commits you do not recognise, that is another agent's work: tell the user and do not rewrite anything to get past it.

If a commit hook refuses your commit because of someone else's files, do not bypass the hook. Treat it as an overlap (below).

## When your lane and theirs overlap

An overlap is when your task needs a file that another agent has changed, or another agent has changed lines you wrote. Signs of it: a file you need to edit has changes in it that are not yours; an edit fails because the file changed under you; your own change is gone or altered.

Do not resolve it yourself. Do not overwrite their change, merge the two by guesswork, or build on top of a version that is still moving. Instead:

1. **Finish and commit everything else in your lane** that the overlap does not touch.
2. **Judge whether the other agent is still working on it.** Changes that are uncommitted, or a file that changes again when you look a little later, mean they are.
3. **If they are still working, wait.** Check back a few times over a few minutes. Once their change is committed and the file has stopped moving, read it again, and if your change still fits on top of theirs, make it and carry on.
4. **Otherwise, stop and tell the user.** This applies when the waiting does not end, or when their change and yours cannot both stand. Say that you stopped because of an overlap with another agent's work, and name the file, what they changed, what you needed to change, and what you have already committed. Then wait for instructions.

Stopping here is the right outcome, not a failure. An unfinished task with a clear note costs the user a minute; two agents overwriting each other costs them the work.

## Reporting

Your final report covers your work only. Mention other agents' work only where it affected yours: a check that failed because of it, an overlap you waited out, or an overlap that stopped you.
