+++
title = "A Lesser-Known Git Hook"
date = 2026-02-13
+++

`commit-msg`

Is just writing "`commit-msg`" a good hook for a note? Probably not. Is it a good hook for git? That's unclear, but it certainly is a git hook!

If you've ever used [git hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks), you've probably used [`pre-commit`](https://pre-commit.com). But there are also a bunch of other, less useful hooks.

Some of them are nice; `prepare-commit-msg` lets you supply a starting commit message when someone goes to write it.

You've also got `commit-msg`, the subject of today, which runs _after_ you've provided a commit message. This hook can reject your commit message, or even edit it secretly! I don't know why anyone would do the latter, but we can try.

# A Commit Hook

To write a hook, create an executable script in `.git/hooks`. The script has to have the same name as the hook, so our `commit-msg` hook must be `.git/hooks/commit-msg` and a `pre-commit` hook has to be `.git/hooks/pre-commit`. Unfortunately, this means you can only have one hook per type.

```sh,name=.git/hooks/commit-msg
#!/bin/bash

case $((RANDOM % 10)) in
	0)
		echo -e "\n\nHelp, I'm trapped in a git commit!" >> "$1"
		;;
	1)
		echo -e "\n\nYou found me!" >> "$1"
		;;
	2)
		echo -e "\n\nThanks for reading the full message." >> "$1"
		;;
	3)
		echo -e "\n\nLinus Torvalds didn't invent git so you could mess with commit messages like this." >> "$1"
		;;
esac
```

This script will append to the commit message 40% of the time, falling back to do nothing in the unspecified default to the [`case`](https://www.gnu.org/software/bash/manual/html_node/Conditional-Constructs.html#index-case) statement.

Some quick notes on this script:
1. I'm using `echo`'s `-e` flag, which is not strictly [POSIX compliant](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/echo.html). You can use `printf` instead.
2. Make sure to have the `(( ))` around `RANDOM`. It tells bash to do [math](https://www.gnu.org/software/bash/manual/html_node/Shell-Arithmetic.html) rather than interpreting `%` for [job control](https://www.gnu.org/software/bash/manual/html_node/Job-Control-Basics.html).
3. `RANDOM` is bash's [random number generator](https://www.gnu.org/software/bash/manual/html_node/Bash-Variables.html#index-RANDOM).

Make sure it's executable with
```sh
chmod +x .git/hooks/commit-msg
```

Otherwise, git will just ignore the hook and move forward with the commit.
```
hint: The '.git/hooks/commit-msg' hook was ignored because it's not set as executable.
hint: You can disable this warning with `git config advice.ignoredHook false`.
```

# Try It Out

Now, let's see how it works! Let's generate a quick git history.

```sh
echo 1 > FILE
git add FILE
git commit -m First
echo 2 > FILE
git add FILE
git commit -m Second
echo 3 > FILE
git add FILE
git commit -m Third
```

When git prints the first line of the commit message, everything looks normal.
```
[master d20446d] Third
 1 file changed, 1 insertion(+), 1 deletion(-)
```

The added message is hidden by the two newlines at the start! Try a `git log` and you should see some added messages sneaking in there.
```
Third

Help, I'm trapped in a git commit!
```

You will likely have different messages, since they're random. About 20% of the time, or $0.6^3$, you won't have any added messages in three commits.

# Closing Notes

Please don't put this script anywhere. The whole business of editing in `commit-msg` is probably best left alone. You can offer people pre-filled starting templates for their commit messages in `prepare-commit-msg`, then validate with a `commit-msg` that rejects messages. The silent changes seem prone to causing mistakes that won't be caught immediately.
