---
layout: post
title: "How to call bash(not shell) from ruby"
date: 2013-09-21 21:53
comments: true
categories: ruby bash shell
---

Few days ago I was writing a ruby wrapper for [SoX](http://sox.sourceforge.net/)
command line tool. To reduce disk IO I wanted to use [process substitution](http://en.wikipedia.org/wiki/Process_substitution).
It's a cool shell feature which allows to use command output as an input file for another command.
It's pretty useful if the second command doesn't work with standard input or you need
to pass more than 1 input.

Let me show the classic example(works in bash and zsh):

```bash
cat <(echo 'Saluton!') <(echo 'Kiel vi fartas?')
# => Saluton! Kiel vi fartas?
```

So statement `<(echo 'Saluton!')` is treated like a file which contains line `Saluton!`.
Underhood bash(zsh) creates a named pipeline where output of `echo 'Saluton!'` is written.
Then the named pipeline is passed to `cat` command.

You can see it:

```bash
echo  <(echo 'Saluton!')
# => /dev/fd/63
```



So I wanted to use it in ruby:
```ruby
cmd = "cat <(echo 'Saluton!') <(echo 'Kiel vi fartas?')"
system(cmd)
```

But unfortunately it doesn't work:
```
sh: 1: Syntax error: "(" unexpected
```


The problem is that ruby's `system` method and back quotes use`sh`
not your current shell (which in my case is `bash`).

```ruby
system "echo $0"
# => sh
```

In shells `$0`points to the current script or to interpreter if you're running it interactively.


Fortunately there is a way to create a workaround to run bash:

```ruby
require 'shellwords'

def bash(command)
  escaped_command = Shellwords.escape(command)
  system "bash -c #{escaped_command}"
end
```

Bash has option `-c` which takes bash script to execute.
[Shellwords](http://www.ruby-doc.org/stdlib-2.0/libdoc/shellwords/rdoc/Shellwords.html)
is a standard ruby library which provides a method to escape shell commands.

So now it works as we want it to be:

```ruby
bash("echo $0")  # => bash
cmd = "cat <(echo 'Saluton!') <(echo 'Kiel vi fartas?')"
bash(cmd)        # => Saluton! Kiel vi fartas?
```

Thanks for reading!
