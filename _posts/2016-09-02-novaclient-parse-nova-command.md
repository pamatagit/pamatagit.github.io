---
layout: post
title: novaclient解析nova命令
date: 2016-08-30
category: "nova"
---

*声明：本文所有分析基于kilo版本。*

解析命令需要用到python的argparse标准库，先看一个例子。

{% highlight python %}

    >>> # sub-command functions
    >>> def foo(args):
    ... print args.x * args.y
    ...
    >>> def bar(args):
    ... print '((%s))' % args.z
    ...
    >>> # create the top-level parser
    >>> parser = argparse.ArgumentParser()
    >>> subparsers = parser.add_subparsers()
    >>>
    >>> # create the parser for the "foo" command
    >>> parser_foo = subparsers.add_parser('foo')
    >>> parser_foo.add_argument('-x', type=int, default=1)
    >>> parser_foo.add_argument('y', type=float)
    >>> parser_foo.set_defaults(func=foo)
    >>>
    >>> # create the parser for the "bar" command
    >>> parser_bar = subparsers.add_parser('bar')
    >>> parser_bar.add_argument('z')
    >>> parser_bar.set_defaults(func=bar)
    >>>
    >>> # parse the args and call whatever function was selected
    >>> args = parser.parse_args('foo 1 -x 2'.split())
    >>> args.func(args)
    2.0
    >>>
    >>> # parse the args and call whatever function was selected
    >>> args = parser.parse_args('bar XYZYX'.split())
    >>> args.func(args)
    ((XYZYX))

{% endhighlight %}

上例首先初始化了一个argpare.ArgumentParser实例，然后为该实例添加了一个subparsers。
该subparsers定义了2个subcommand，foo和bar，为foo命令添加了参数-x、y、func，为bar命令添加了参数z、func，foo命令的func参数默认为foo方法，bar命令的func参数默认为bar方法。

然后通过这个parser实例来解析命令，若命令是foo,则执行foo方法，若命令为bar，则执行bar方法。


novaclient解析nova命令的原理与上例相同。只是nova有那么多个命令，是怎么将这些命令一个一个添加成subcommand的呢？

novaclient是通过下面这个方法来实现的：

{% highlight python %}

    def _find_actions(self, subparsers, actions_module):
        for attr in (a for a in dir(actions_module) if a.startswith('do_')):
            # I prefer to be hyphen-separated instead of underscores.
            command = attr[3:].replace('_', '-')
            callback = getattr(actions_module, attr)
            desc = callback.__doc__ or ''
            action_help = desc.strip()
            arguments = getattr(callback, 'arguments', [])

            subparser = subparsers.add_parser(
                command,
                help=action_help,
                description=desc,
                add_help=False,
                formatter_class=OpenStackHelpFormatter)
            subparser.add_argument(
                '-h', '--help',
                action='help',
                help=argparse.SUPPRESS,
            )
            self.subcommands[command] = subparser
            for (args, kwargs) in arguments:
                subparser.add_argument(*args, **kwargs)
            subparser.set_defaults(func=callback)
{% endhighlight %}

这个方法的actions_module参数是一个module，novaclient中这个module为novaclient.v2.shell，该文件中定义了'do_list'、'do_boot'等方法。

该方法会去该module中查找以'do_'开头的所有方法，将这些方法名去掉开头的'do_'，并以'-'代替'_'后作为subcommand。

然后去获取该subcommand所需的参数，这些参数都以装饰器的形式添加在do_方法中,装饰器将这些参数放在do_方法的arguments属性中。如下：
{% highlight python %}

    @cliutils.arg(
    '--sort',
    dest='sort',
    metavar='<key>[:<direction>]',
    help=('Comma-separated list of sort keys and directions in the form'
      ' of <key>[:<asc|desc>]. The direction defaults to descending if'
      ' not specified.'))
    def do_list(cs, args):
    """List active servers."""
    imageid = None
    flavorid = None
    ...
    ...

{% endhighlight %}

_find_actions方法获取到subcommand的参数后将这些参数添加到subcommand上，然后以"subparser.set_defaults(func=callback)"指定了该subcommand的默认func参数为相应的do_方法。

novaclient解析完命令后会以args.func（）方法调用相应的do_方法。















