---
title: "命令行构建工具 - Thor"
date: 2013-03-09T00:17:18+08:00
---

Thor 是一个用来构建强大命令行的工具包，诸如 Bundler、Rails、Vagrant 这样的项目都在使用它。
## 基本使用

Thor 类会被封装成带有多个命令的 CLI，比如 `git`、`bundler`。
类的公有方法表征命令。

``` ruby
class MyCLI < Thor
  desc "hello NAME", "say hello to NAME"
  def hello(name)
    puts "Hello #{name}"
  end
end
```

有了上面的类，然后调用 `MyCLI.start(ARGV)`，CLI 就可以执行了。

接下来的例子，我会假设当前目录下有一个叫 `my_cli` 的 ruby 源文件，内容大致是这样的。

``` ruby
require "thor"

class MyCLI < Thor
  # contents of the Thor class
end

MyCLI.start(ARGV)
```

如果传递给 `start` 的参数列表是空的，Thor 会打印出 CLI 的帮助信息。
帮助信息会自动使用 CLI 的名称。

``` bash
$ ruby ./my_cli
Commands:
  my_cli hello NAME      # say hello to NAME
  my_cli help [COMMAND]  # Describe available commands or one specific command
```

带参数执行 `hello` 命令，就会调用 Thor 类中相应的方法。

``` bash
$ ruby ./my_cli hello world
Hello world
```

如果没有带参数，Thor 会打印出错误信息。

``` bash
$ ruby ./my_cli hello
ERROR: my_cli hello was called with no arguments
Usage: "my_cli hello NAME".
```

当然，也可以在 ruby 代码中使用可选参数，这样 CLI 中的参数就变成可选的了。

``` ruby
class MyCLI < Thor
  desc 'hello NAME', 'say hello to NAME' # usage, description
  def hello(name, from = nil)
    puts "from: #{from}" if from
    puts "Hello #{name}"
  end
end
```

然后再执行时，

``` bash
$ ruby ./my_cli hello world
Hello Yehuda Katz

$ ruby ./my_cli hello world "Windor C"
from: Carl Lerche
Hello Yehuda Katz
```

这种方法在某些情况下会有用，不过，大多数情况，你可能会更偏向于使用 **Unix风格的选项**。
## 长描述

Thor 默认会使用 `desc` 提供的简短描述来充当一个命令的详细描述。

``` bash
$ ruby ./my_cli help hello
Usage:
  my_cli hello NAME

say hello to NAME
```

但普遍做法是直接提供一个长描述。`long_desc` 就是用来做这件事情的。

``` ruby
class MyCLI < Thor
  desc "hello NAME", "say hello to NAME"
  long_desc <<-LONGDESC
    `cli hello` will print out a message to a person of your
    choosing.

    You can optionally specify a second parameter, which will print
    out a from message as well.

    > $ cli hello "Yehuda Katz" "Carl Lerche"

    > from: Carl Lerche
  LONGDESC
  def hello(name, from=nil)
    puts "from: #{from}" if from
    puts "Hello #{name}"
  end
end
```

描述中，多加一个空行来进行换行，类似于 markdown。也可以在行首添加 `\x5` 强制换行。

``` ruby
require 'thor'
class MyCLI < Thor
  desc 'hello NAME', 'say hello to NAME' # usage, description
  long_desc <<-LONGDESC
    `cli hello` will print out a message to a person of your
    choosing.

    You can optionally specify a second parameter, which will print
    out a from message as well.

    > $ cli hello "Yehuda Katz" "Carl Lerche"
    \x5> from: Carl Lerche
  LONGDESC
  def hello(name, from = nil)
    puts "from: #{from}" if from
    puts "Hello #{name}"
  end
end

MyCLI.start(ARGV)
```

**为了保持简短性和可读性，一般会将长描述保存在单独的文件里，然后用`File.read`从中读取所需内容。**
## 选项
### 基本使用

在 Thor 中，为命令添加选项是一件非常简单的事情。

``` ruby
class MyCLI < Thor
  desc "hello NAME", "say hello to NAME"
  option :from
  def hello(name)
    puts "from: #{options[:from]}" if options[:from]
    puts "Hello #{name}"
  end
end
```

这样，`--from` 选项就可以使用了。

``` bash
$ ruby ./my_cli hello --from "Windor C" world
from: Windor C
Hello world

$ ruby ./my_cli hello world --from "Windor C"
from: Windor C
Hello world

$ ruby ./my_cli hello world --from="Windor C"
from: Windor C
Hello world
```
### 指定选项类型

选项默认的都是字符串类型，可以使用 `:type` 为选项指定其他类型。

``` ruby
class MyCLI < Thor
  option :from
  option :yell, :type => :boolean
  desc "hello NAME", "say hello to NAME"
  def hello(name)
    output = []
    output << "from: #{options[:from]}" if options[:from]
    output << "Hello #{name}"
    output = output.join("\n")
    puts options[:yell] ? output.upcase : output
  end
end
```

现在你可以大声的向世界问好了。

``` bash
$ ruby ./my_cli hello  --yell world --from "Windor C"
FROM: WINDOR C
HELLO WORLD

$ ruby ./my_cli hello world --from "Windor C" --yell
FROM: WINDOR C
HELLO WORLD
```
### 指定选项为必需选项

用 `:required` 指定一个选项为必需选项。

``` ruby
class MyCLI < Thor
  option :from, :required => true
  option :yell, :type => :boolean
  desc "hello NAME", "say hello to NAME"
  def hello(name)
    output = []
    output << "from: #{options[:from]}" if options[:from]
    output << "Hello #{name}"
    output = output.join("\n")
    puts options[:yell] ? output.upcase : output
  end
end
```

如果不带上 `--from` 选项，命令就会出错。

``` bash
$ ruby ./my_cli hello world
No value provided for required options '--from'
```
### 配置选项

以下是可以提供给选项的所有元信息。
- `:desc`，选项的具体说明。使用 `cli help hello`查看命令的使用方法时，说明文字会跟在选项后面一起出现。
- `:banner`，出现在选项使用描述中的简短占位符。默认的是选项的大写， `[--from=FROM]`。
- `:required`，表示该选项必须要出现。
- `:default`，选项的默认值。一个选项不能既有 `:required`，又有 `:default`。
- `:type`， `:string`、`:hash`、`:array`、`:numeric`、`:boolean`。
- `:aliases`，选项的一系列别名。一般来说，都会用别名来提供更简短的选项。

如果只是想配置选项的类型，那么可以用 `options` 一次性添加多个选项。
上面的例子就可以这样写（可以直接用 `required` 表示一个必需的 `string` 类型）。

``` ruby
class MyCLI < Thor
  desc "hello NAME", "say hello to NAME"
  options :from => :required, :yell => :boolean
  def hello(name)
    output = []
    output << "from: #{options[:from]}" if options[:from]
    output << "Hello #{name}"
    output = output.join("\n")
    puts options[:yell] ? output.upcase : output
  end
end
```
## 类选项

如果需要为所有命令提供选项（比如说：--verbose），可以使用 `class_option`。
类选项是应用在 CLI 所有命令上的，
除此之外，类选项和单个命令的选项没有其他任何区别。

ruby 代码中，提供给方法的 `options` 哈希表包含了类选项。

``` ruby
class MyCLI < Thor
  class_option :verbose, :type => :boolean

  desc "hello NAME", "say hello to NAME"
  options :from => :required, :yell => :boolean
  def hello(name)
    puts "> saying hello" if options[:verbose]
    output = []
    output << "from: #{options[:from]}" if options[:from]
    output << "Hello #{name}"
    output = output.join("\n")
    puts options[:yell] ? output.upcase : output
    puts "> done saying hello" if options[:verbose]
  end

  desc "goodbye", "say goodbye to the world"
  def goodbye
    puts "> saying goodbye" if options[:verbose]
    puts "Goodbye World"
    puts "> done saying goodbye" if options[:verbose]
  end
end
```
## 子命令

当 CLI 变得越来越复杂时，一个命令可能有它自己的子命令集合。
`git remote` 就是这样一个例子。
`git` 是一个 CLI，`remote` 是 `git` 的一个命令。
而 `remote` 又有 `add`、`rename`、`rm`、`prune`、`set-head` 等等多个子命令。

在 Thor 中，可以创建一个新的 Thor 类代表子命令集合，然后在逻辑上的父类中定义一个映射指向这个新的类。

下面是 `git remote` 的一个简易实现。

``` ruby
module GitCLI
  class Remote < Thor
    desc "add <name> <url>", "Adds a remote named <name> for the repository at <url>"
    long_desc <<-LONGDESC
      Adds a remote named <name> for the repository at <url>. The command git fetch <name> can then be used to create and update
      remote-tracking branches <name>/<branch>.

      With -f option, git fetch <name> is run immediately after the remote information is set up.

      With --tags option, git fetch <name> imports every tag from the remote repository.

      With --no-tags option, git fetch <name> does not import tags from the remote repository.

      With -t <branch> option, instead of the default glob refspec for the remote to track all branches under $GIT_DIR/remotes/<name>/, a
      refspec to track only <branch> is created. You can give more than one -t <branch> to track multiple branches without grabbing all
      branches.

      With -m <master> option, $GIT_DIR/remotes/<name>/HEAD is set up to point at remote's <master> branch. See also the set-head
      command.

      When a fetch mirror is created with --mirror=fetch, the refs will not be stored in the refs/remotes/ namespace, but rather
      everything in refs/ on the remote will be directly mirrored into refs/ in the local repository. This option only makes sense in
      bare repositories, because a fetch would overwrite any local commits.

      When a push mirror is created with --mirror=push, then git push will always behave as if --mirror was passed.
    LONGDESC

    option :t, :banner => "<branch>"
    option :m, :banner => "<master>"
    options :f => :boolean, :tags => :boolean, :mirror => :string
    def add(name, url)
      # implement git remote add
    end

    desc "rename <old> <new>", "Rename the remote named <old> to <new>"
    def rename(old, new)
    end
  end

  class Git < Thor
    desc "fetch <repository> [<refspec>...]", "Download objects and refs from another repository"
    options :all => :boolean, :multiple => :boolean
    option :append, :type => :boolean, :aliases => :a
    def fetch(respository, *refspec)
      # implement git fetch here
    end

    desc "remote SUBCOMMAND ...ARGS", "manage set of tracked repositories"
    # pay attention to this line
    subcommand "remote", Remote
  end
end
```

