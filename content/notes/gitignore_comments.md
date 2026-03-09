+++
title = "Comments are Weird in Gitignores"
date = "2026-03-08"
description = "Just a weird public service announcement that .gitignore files are strict about comments. The # has to come first, otherwise it matches files. No leading whitespace, no inline comments."
+++

For the most part, it's hard to get simpler than a `.gitignore` file. You have files, patterns, and comments that start with a `#`.

I recently learned that the comment requirement is much stricter than I expected! In almost every other case, `#` comments can come at any point and go until the end of the line. But not in `.gitignore` files. Instead, the comment [has to start](https://git-scm.com/docs/gitignore) at the start of the line. No inline comments for `.gitignore`!

I'd rather show than tell, so let's get into it.

# Checking What's Ignored

To really see this, we'll need to be able to check what git ignores. You can [check](https://git-scm.com/docs/git-check-ignore) whether or not a file is covered by the `.gitignore` using the `git check-ignore` command.

Let's say you have `file1` and `file2`, and your `.gitignore` is:
```name=.gitignore
file1
```

Run the check:
```sh
git check-ignore file1 file2
```

And you should just see the ignored file:
```
file1
```

The other file, `file2`, isn't ignored by git so it isn't printed.

# Leading Whitespace

First off, you can't have any leading whitespace before your comment, like in this `.gitignore`.

```name=.gitignore
# Comment
 # After a space
```

Let's check what's covered! Note that you need quotes to get the space in there right. "` # After a space`" is not the same as "`# After a space`".

```sh
git check-ignore "# Comment" " # After a space"
```

You should see the second line of the `.gitignore` matching, while the first line is correctly treated as a comment.

```
 # After a space
```

If you, for some terrible reason, had a file called "` # After a space`" then git would ignore it. (If you also had to ignore a file starting with a `#`, truly cursed, you'd use an escape, `\#`.)

# Comments After Files

I would have expected a comment after a file to be okay. It usually is, right? Nope.

```name=.gitignore
file # After a space
```

Unfortunately, git doesn't see a comment here. Instead, it thinks you're worried about the ridiculous file "`file # After a space`".

```sh
git check-ignore file "file # After a space"
```

This check will only find the "`file # After a space`" file as ignored, _not_ `file`.

# Consequences

Really, we have two problems.
1. Git could interpret a comment as a file if the line starts with a space.
2. Git doesn't allow a filename with a comment on the same line.

I think the first problem is okay; you're never going to have a filename that either starts with whitespace or contains a "#" in its name. If you do, you deserve git messing with you.

The second problem could be major. You might think a file is ignored, but git adds it happily. Since you felt the need to comment the line, I bet that there are some files that match the rule, and this problem isn't an edge case. We probably should have better tooling for `.gitignore` to catch that gotcha!
