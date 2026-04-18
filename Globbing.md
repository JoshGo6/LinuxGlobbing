---
tags:
  - expansion
  - glob
  - shopt
  - configuration
---
# Globbing

This article contains the following topics:

- [Basics of Globbing](#Basics%20of%20Globbing)
- [Modifying globbing behavior with `shopt` and `set`](#Modifying%20globbing%20behavior%20with%20`shopt`%20and%20`set`)
- [Examples using `**/` syntax](#Examples%20using%20`**/`%20syntax)
- [See also](#See%20also)


## Basics of Globbing

The following glob patterns in bash perform filename expansions:

> [!NOTE] 
> These patterns are *not* the same as the corresponding regex patterns:

- `?`: Exactly one character
- `*` Zero or more characters. This will **not** match strings that start with `.` unless `shopt -s dotglob` is set.
- `[afg].txt`. Matches `a.text`, and `f.txt` and `g.txt`
- `[a-d].txt`. Matches `a.text`, and `b.txt`, `c.txt` and `d.txt`
-  `[^a-d]*.txt`. Will match any text file except one starting with an `a`, `b`,  `c`, or `d`.
- `{b*,c*,*est*}.txt`. Matches `b.txt`, `central.txt`, and `hester.txt`. Do not put spaces between items.
- `**/` Matches zero, one, or many directories. The command `git add ./**/*.md` stages all Markdown files to any depth. Not all shells or utilities understand the `**/` syntax for recursive matching.

> [!IMPORTANT]
> To use `**/` syntax, first enable it using `shopt -s globstar`. Additionally, parameter expansions don't occur inside single or double quotes. For more complete information, see [Shell Expansion and Quoting](Shell%20Expansion%20and%20Quoting.md).

When you create a command with unquoted globbing characters, the shell attempts to match filenames using the glob patterns. Depending on what the shell finds, this can influence the output. If the shell successfully matches filenames by expanding the glob, the command that would have received the glob now receives literal filenames. If the shell fails to expand the glob, the command receives the glob intact. The following two shell sessions illustrate this behavior.

In the following shell session, we first use `shopt` to ensure that the shell doesn't use glob matching to match filenames starting with `.` unless there's a literal `.` in the pattern to match on. (For more information on this, see [Modifying globbing behavior with `shopt` and `set`](#Modifying%20globbing%20behavior%20with%20`shopt`%20and%20`set`).) Next, we verify this behavior with `ls`. Finally, we perform a recursive `find` for JSON files and limit the output to four entries.

```shellsession
# Don't match any files starting with a dot (.)
$ shopt -u dotglob
# Verify that there's a JSON file in this directory
$ ls .*.json
.claude.json
# Verify that the JSON file won't be matched unless the glob explicitly starts with a dot.
$ ls *.json
ls: cannot access '*.json': No such file or directory
# The find command finds at least 4 files.
$ find . -name *.json | head -4
./.config/Xmind/Electron v3/vana/state/notification.json
./.config/Xmind/Electron v3/vana/state/app.json
./.config/Xmind/Electron v3/vana/state/account.json
./.config/Xmind/Electron v3/vana/state/keybindings.json
```

In the previous output, `*.json` was passed literally to `find`. In other words, glob expansion did *not* occur on the `*`. Now let's change the `dotglob` behavior to match files starting with a `.` even in situations where the pattern does not explicitly start with a `.`, which, in this case, causes the output of both `ls` and `find` to change:

```shellsession
# Now globbing can match files starting with a dot, even if the pattern doesn't start with a dot.
$ shopt -s dotglob
# We verify that * matches a dot.
$ ls *.json
.claude.json
$ find . -name *.json | head -4
./.claude.json
```

Because `*.json` now matches `.claude.json`, the argument `*.json` *does* undergo filename expansion. In other words, now that we changed the globbing behavior via `shopt -s dotglob`, the shell successfully replaces the glob `*.json` with `.claude.json`, and the final command becomes `find . -name .claude.json | head -4`, since filename expansion occurs *before the command is ever run*, and we only get one result for `find`.

## Modifying globbing behavior with `shopt` and `set`

You can use the shell `shopt` command to modify shell behavior. There are numerous behaviors that you can set or unset, depending on your needs. There are `shopt` settings and `set` settings, but you can use `shopt` to enable or disable either type.

```bash
# Enable a shopt setting
shopt -s <setting>

# Disable a shopt setting
shopt -u <setting>

# Enable a set setting
shopt -s -o <setting>

# Disable a set setting
shopt -s -o <setting>

# List all shopt settings and whether they're set or not
shopt -p

# Search for an option
shopt -p | grep globstar

# Completely disable filename globbing
set -f

# Reenable filename globbing
set +f
```

The following are some `shopt` settings that pertain to globbing:

- `nocaseglob`: Use case-insensitive matching with filenames.
- `nullglob`: Filename expansion patterns that match no files expand to nothing and are removed, rather than expanding to themselves.
- `dotglob`: Match filenames starting with `.`, except for the `.` and `..` filenames.
- `globstar`: If set, the pattern `**` without a trailing `/` used in a filename expansion matches all files and zero or more directories and subdirectories. The pattern `**/` matches only directories and subdirectories.

## Examples using `**/` syntax

As mentioned previously, to use the `**` syntax, you first need to enable it by running `shopt -s globstar`. This globbing pattern behaves according to the following rules:

- `**` without a trailing `/` matches all files and zero or more directories.
-  `**/` (that is, with a trailing `/`) only matches zero or more directories. It does not match any files.

> [!IMPORTANT]
> The `*` itself can match 0, 1, or multiple characters, which means that if it's the last glob character  in a directory that contains subdirectories, depending on what preceded it in the pattern, it can match all files and subdirectories to a depth of 1.

We'll demonstrate this behavior in the code examples that follow, which operate on a directory tree with the following structure:

```shellsession
$ tree
.
├── 0-1.md
├── 0-1.txt
├── level1
│   ├── 1-1.txt
│   └── level2
│       ├── 2-1.md
│       ├── 2-1.txt
│       └── 2-2.txt
└── non-text-file.xls
```

By using `ls` with different glob patterns, you can find different files and directories, as the following commands show:

> [!note]
> When the argument to `ls` matches a directory, `ls` doesn't list that directory name. It lists the directory's contents. When you see a directory name listed as output, this is due to the fact that the parent of that directory was matched.

```shellsession

# List all files and directories in the current directory, non-recursively
$ ls *
0-1.md  0-1.txt  non-text-file.xls

level1:
1-1.txt  level2

# Because of the trailing /, this doesn't match 0-1.txt in the current directory.
# It only matches level1 directory.
$ ls */
1-1.txt  level2

# List text files in the current directory
$ ls *.txt
0-1.txt

# List text files in subdirectories to any depth but not the current directory
$ ls */**/*.txt
$ ls */**/*.txt
level1/1-1.txt  level1/level2/2-1.txt  level1/level2/2-2.txt

# List all files and directories in this directory and all subdirectories
$ ls **/*
0-1.md  0-1.txt  level1/1-1.txt  level1/level2/2-1.md  level1/level2/2-1.txt  level1/level2/2-2.txt  non-text-file.xls

level1:
1-1.txt  level2

level1/level2:
2-1.md  2-1.txt  2-2.txt

# List text files in current directory and subdirectories 
$ ls **/*.txt
0-1.txt  level1/1-1.txt  level1/level2/2-1.txt  level1/level2/2-2.txt

# List all markdown and XLS files, in this and all subdirectories
$ ls **/*.{md,xls}
0-1.md  level1/level2/2-1.md  non-text-file.xls
```

## See also

- [Shell Expansion and Quoting](Shell%20Expansion%20and%20Quoting.md)
- [Globbing with `find` and `grep`](../Finding%20files/Globbing%20with%20`find`%20and%20`grep`.md)
- [Globbing](https://tldp.org/LDP/abs/html/globbingref.html)
- [The Shopt Builtin](https://www.gnu.org/software/bash/manual/html_node/The-Shopt-Builtin.html)
- [Wildcards](https://tldp.org/LDP/GNU-Linux-Tools-Summary/html/x11655.htm)
- [The Set Builtin](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html)
