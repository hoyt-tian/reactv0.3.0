# JSX解析

严格来讲，jsx的语法解析并不是React自己实现的，而是借助了第三方的工具库esprima，[http://esprima.org/](http://esprima.org/)，它是从jQuery中衍生出来的一个ECMAScript解析库，更多信息可以查阅[https://github.com/jquery/esprima](https://github.com/jquery/esprima)。

在./bin/jsx文件中，可以看到编译过程的处理脚本。这一部分内容不属于React框架运行环境的部分，而且之后被babel取代了，就不再介绍了，感兴趣的读者可以自行研究esprima的使用。


