title: 如何利用webhook搭建自动化部署 - ghostsf
categories: 技术栈
tags: [webhook,自动化部署]
date: 2017-02-21 03:47:54
---
首先,我们明确自己的需求:
搭建一个自动化部署平台,其需求如下:

 - 能做到自动拉取代码
 - 自动编译
 - 自动更新数据库表结构
 - 只更新master分支

这是一个最简单的自动化部署平台需求，下面就来看看怎么实现，需要说明的是,因为是最简单的方案，故没有考虑分布式的架构。
 我们来做功能拆分，拆分如下：
**代码同步模块:**

正常代码拉取
强制推送代码拉取
忽略其他分支的代码推送
编译模块

正常编译
异常编译回滚
部署模块

编译文件自动部署
重启服务端
重启代理工具

**日志模块：**

正常情况下日志输出
代码拉取失败日志输出
编译错误日志输出
服务端重启日志输出

现在功能需求定好了，开始进行代码编写，因为这是一个最简单的部署，所以数据量非常小，用文本文件足以，故这里不采用数据来增加系统复杂程度
首先，我们我们要写一个接口，以便于接收由webhook发出的数据，由于webhook的数据发送方式是post，故需要一个接收post数据的地址。例如：
http://xxx.xxx.com/xxx/xxx/git/post.php 这样一个地址，当然，这个php文件具体怎么写就不知道了哈，他的作用只有一条：调用执行预定脚本： auto.sh并将收到的参数传递给脚本
然后我们剩下的工作就是写auto.sh这个脚本了，我们所有的工作都需要在auto.sh这个脚本上完成。
首先，我们来搞定自动拉取代码

    git pull origin master

这是正常拉取代码的，但是这条命令需要拉取的分支不存在强制推送状态，故这条命令适用性不够广泛，所以我们要改造一下，改成下面的样子：

    git reset --hard <commit id>
    git pull origin master
    puts “时间 拉取代码成功，分支为master” >> "pull.txt"

然后我们就搞定了代码拉取的问题，设定好commit id 任你分支怎么变化怎么强推我都能正常拉取代码。为了正常判断状态，我们将这段代码新建个脚本，命名为 pull_code.sh 然后交给auto.sh调用，这样我们就可以在auto中写上判断条件来判断脚本执行成功还是失败，这样便于日志输出
实际上，这里只做到啦拉取master分支的代码，我们可以将这段代码改改，将分支名作为参数，然后加上checkout语句，就可以拉取任意分支了，比如这样：

    git checkout 分支名（参数，外部传递）
    git reset --hard <commit id>
    git pull origin  分支名

至于commit id 从配置文件中读出来就行了，然后就变成了可以部署其他分支的代码了。到这里，拉取代码的部分就算完成了。
接下来是自动编译部分，同样，基于扩展性需求，我们可以将它做成一个脚本，已编译rails应用为例，我们一般需要编译资源执行如下命令：

    bundle exec rake assets:precompile RAILS_ENV=production

所以我们将这条命令写成脚本，命名为：compile.sh 同样，编译部分就完成了，其他语言按照其他语言格式将执行的命令写入到脚本中去，然后等待auto.sh的调用即可

**部署模块**同样的操作手法，将需要执行的命令一条一条写入脚本中，等待被调用
然后我们开始写最重要主文件，也就是auto.sh脚本（注：以下是伪代码，不具有执行能力，仅提供思路参考）

**以下是伪代码：**

    开始接受参数：params
    解析参数，需要：分支信息 commit信息
    根据分支信息和commit信息判断是否需要更新，如果无需更新，结束
    需要更新，开始切换用户，获取更新权限，写入参数到文本文件中，避免因切换参数导致参数丢失
    从文本文件中读取参数，调用pull_code.sh脚本，同时传递分支参数
    接收pull_code.sh脚本执行结果，开始根据执行结果打印日志，同时保存执行结果到变量code
    根据变量code的值判断代码拉取是否成功，判断是否需要编译资源，同时打印日志
    调用部署脚本，开始执行部署操作
    日志部分开始工作，对每次执行结果进行记录
    汇总日志部分，开始分析是否可以重新启动服务（热启动或其他）
    开始清理除日志之外的中间文件，清理工作目录
    执行完成

以上就是一个最简单的利用webhook进行自动化部署的案例，其原理是先用钩子触发回调，拿到更新信息，然后调用预先定义好的自动化部署脚本进行部署，最后完成操作
