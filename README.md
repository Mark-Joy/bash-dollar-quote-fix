# Introduction

This repo contains my fix of bash's long standing issue ["Variable dollar sign is escaped by bash-completion"](https://github.com/scop/bash-completion/issues/290). Be warned that this is user code, so bug may happen.

# Usage
To apply the fix, bash has to be complied from source.

+ Download [bash-5.1.8.tar.gz](https://ftp.gnu.org/gnu/bash/bash-5.1.8.tar.gz) from [gnu/bash](https://ftp.gnu.org/gnu/bash/). Extract its source code to ~/bash_src/
+ Replace bashline.c in ~/bash_src/
+ Replace readline.h and complete.c in ~/bash_src/lib/readline/
+ Replace shquote.c in ~/bash_src/lib/sh/

Readline source code was modified, so bash project has to be configured **WITHOUT** `--with-installed-readline`.
Below is an example on how to compile bash.
```Python
cd ~/bash_src/

./configure --prefix=/usr                 \
        --bindir=/bin                     \
        --htmldir=/usr/share/doc/bash     \
        --without-bash-malloc

make clean

make
```
Refer https://stackoverflow.com/questions/21644870/how-to-compile-bash for more information.

To make sure that replaced files are taking effect, do `make clean` before `make`.

After **bash** file is compiled successfully: 
+ Replace **bash** file in /usr/bin/
+ Replace **bash_completion** file in /usr/share/bash-completion/

# Original Issue
Suppose we have folders named `test space` and `$Name` in home directory. Folder `$dollar` is inside of `$Name`
```Python
ls "$HOME/test space/<TAB> will give
ls "\$HOME/test space/
```
```Python
ls "$HOME/\$Name/<TAB> will give
ls "\$HOME/\$Name/\$dollar
```
```Python
ls $HOME/test\ space/<TAB> will give
ls \$HOME/test\ space/
```
In bash code, **rl_filename_completion_function** function will check for characters that need to be dequoted such as space, tab, newline, colon (;), semicolon (;), dollar symbol ($), At sign (@), etc... 
```python
Full list: " \t\n\\\"'@<>=;|&()#$`?*[!:{~"
```
```Python
[$HOME/test space/] is dequoted to [$HOME/test space/]
[$HOME/test\ space/] is dequoted to [$HOME/test space/]
[$HOME/\$Name/] is dequoted to [$HOME/$Name/]
```
Then it is passed to **bash_directory_completion_hook** where it is expanded in **expand_prompt_string**.
```Python
[$HOME/test space] is expanded to [data/data/com.termux/files/home/test space]
[$HOME/$Name/] is expanded to [data/data/com.termux/files/home/$Name/]
```
Then bash generates matches by looking into the expanded directory for files and folders. In our case, we have a match `[$HOME/$Name/$dollar]`.

Then again, if there is any character needs to be quoted, the result is quoted and display on prompt string.

# How the Fix was made
Simple, during **rl_filename_completion_function**, just neither quote nor dequote dollar symbol for user input. Only quote dollar symbol for generated matches.
In our case `[$HOME/test space/], [$HOME/test\ space/], [$HOME/$Name/]` are the user inputs. `[$dollar]` is bash's generated match.
Follow the above mentioned logic, we have the results: 
```Python
ls "$HOME/test space/<TAB> will give
ls "$HOME/test space/
```
```Python
ls "$HOME/\$Name/<TAB> will give
ls "$HOME/\$Name/\$dollar/
```
```Python
ls $HOME/test\ space/<TAB> will give
ls $HOME/test\ space/
```
For it to works correctly, a fews changes were also made in **bash_directory_completion_hook** and **expand_prompt_string** to control how a given path is expanded.

Aside from fixing the original issue, this patch also preserves the quoting form.

Original bash:
```Python
ls "test space/"<TAB> will give
ls test\ space/
```
Modified bash:
```Python
ls "test space/"<TAB> will give
ls "test space/"
```

# Changes in bash_completion file
```Shell
diff -uNr bash_completion.orig bash_completion                         --- bash_completion.orig        2021-10-22 22:25:21.000000000 +0700
+++ bash_completion     2021-11-10 10:59:09.204587448 +0700
@@ -570,6 +570,11 @@
     local -a toks
     local reset arg=${1-}

+       if [[ $cur == \'* || $cur == \"* ]]; then
+       # Leave out first character
+       cur="${cur:1}"
+    fi
+
     if [[ $arg == -d ]]; then
         reset=$(shopt -po noglob)
         set -o noglob
@@ -578,8 +583,8 @@
         $reset
         IFS=$'\n'
     else
-        local quoted
-        _quote_readline_by_ref "${cur-}" quoted
+        local quoted="$cur"
+        ##_quote_readline_by_ref "${cur-}" quoted

         # Munge xspec to contain uppercase version too
         # https://lists.gnu.org/archive/html/bug-bash/2010-09/msg00036.html
```
There is no need to use **_quote_readline_by_ref** in **_filedir**\
These changes works for both original and modified bash.
