---
layout: post
title:  "Redis 6 ACL(Translated from Redis Lab)"
date:   2020-01-14 12:58:29
categories: Redis
---

Translated from： https://redis.io/topics/acl

Redis访问控制列表(ACL),是一项可以实现限制特定客户端连接可执行命令和键访问的功能，它的工作方式是：客户端连接服务器以后，客户端需要提供用户名和密码用于验证身份：如果验证成功，客户端连接会关联特定用户以及用户相应的权限。Redis可以配置新的客户端连接自动使用default用户进行验证（默认选项），因此配置default用户权限
会使没有验证的客户端只能使用一小部分功能。
Redis 6版本(第一个支持ACL的版本)的默认配置和之前版本完全相同，即每一个新的客户端连接有权限去访问所用命令和键，因此ACL功能对旧版客户端和应用是向后兼容(backward compatible)的，并且对旧版配置用户密码的方式，使用requirepass配置选项是完全支持的，但是不同的是requirepass配置选项只是设定default用户的密码。
Redis AUTH命令在Redis 6版本进行扩展，现在可以使用两个参数形式：

`AUTH <username> <password>`

如果使用旧版本的使用方式：

`AUTH <password>`

会使用 default用户进行验证，所以用这种形式进行验证意味着我们想使用default用户进行身份验证，这种方式可以提供完美的向后兼容旧版本Redis的支持。

##ACL 的使用场景：

在使用ACL之前你需要问一下自己使用这层访问控制的目的是什么。正常来说ACL可以实现下面两个重要目标：
1.	你希望限制用户访问命令和键以提高安全性，因此不在信任列表里的用户没有权限访问，而在信任列表里的用户有可以完成工作的最小访问权限。例如一些客户端只可以执行只读的命令。
2.	你希望提高运营安全， 当程序出错或人为原因操作失误， 进程或用户不允许访问Redis，以避免数据或者配置受到损坏。 例如一个用于拿取延迟工作的工作客户端不应该有权限去执行flushall命令。

另外一个ACL的应用场景是关于管理Redis实例。Redis通常被作为内部托管服务提供给公司内部用户或者作为云服务提供商配置的软件服务。在这两种设置方式中我们必须确保配置命令对用户来说是不可见的。过去版本的redis是暂时通过命令重命名(command renaming)这种方式实现的,是旧版本没有ACL支持的一种临时措施，但是却不是完美的解决方案。


##使用ACL命令配置ACL

ACL 是用DSL(domain specific language)语言来定义用户是否有权限访问特定资源或操作的。每一条ACL规则都是从第一个到最后一个，从左至右来实现的，因为在一些时候，定义规则的顺序会决定用户是否有权限去访问特定资源和操作。

默认配置下只有一个用户被定义（default用户）。我们可以使用 ACL LIST命令来检查当前有效的ACL规则。使用ACL LIST确认刚启动的，使用默认配置的Redis 实例ACL规则如下：

`> ACL LIST`
`1) "user default on nopass ~* +@all"`

上面的命令遵循和Redis配置文件同样的格式返回当前ACL配置的规则。
每一行最前面的两个词是”user”和用户名。之后的词是具体ACL定义的规则。 我们下面将会看到怎样使用这些规则，但是现在，可以简单理解为默认（default）用户的配置是开启的（active（on）），不需要密码（require no password（nopass）），可以访问所有键（to access every possible key（～*))和可以执行任何命令（call every possible command（+@all））的。
另外，对于默认（default）用户来说，没有配置密码规则意味着新的客户端连接会自动使用默认（default）进行验证而不用显式的执行AUTH命令。

##ACL规则
下面是有效的ACL规则，特定规则只是用于启用或删除特定ACL标志的词，或者用于改变用户当前的ACL规则。其他的规则是char 类型的前缀和命令或命令类别的名字，或者键的模式。

##启用和禁止用户
`on`：启用用户：可以使用这个用户进行验证。
`off`：禁止用户：不能使用这个用户进行验证，但已经验证的客户端连接仍然可以继续工作，需要注意的是如果默认（default）用户如果被禁止，默认用户配置将不起作用，新的客户端连接将会不被验证并且需要用户显式发送AUTH命令或HELLO命令进行验证。

##允许和禁止命令
`+<command>`: 添加命令到用户允许执行命令列表。
`-<command>`: 从用户允许执行命令列表删除命令。
`+@<category>`: 允许用户执行所有定义在category类别中的命令。有效的类别例如@admin, @set, @sortedset等等。你可以通过使用ACL CAT命令查看所有预定义的类别，另外一个特殊的类别+@all意思是所有在当前系统里的命令，以及将来在Redis模块中添加的命令。
`-@<category>`: 类似于+@<category> ，但是是从用户可以执行的命令列表中删除特定的命令。
`+<command>|subcommand`:允许 用户启用一个被禁止命令的子命令，请注意这个命令不允许使用-规则，例如-DEBUG|SEGFAULT，只能使用+规则。如果整个命令已经被启用，单独启用子命令没有意义，Redis会返回错误。
`allcommands`: +@all的别名. 注意这个会使用户允许执行所有从Redis模块中添加的命令。
`nocommands`:  -@all的别名。

##允许和禁止特定的键
`~<pattern>`:添加一个键模式以被用在用户执行命令里面。例如~*允许使用所有的键。键的模式是通配符模式，像KEYS命令一样。另外使用多个模式也是被允许的，* allkeys:是~*的别名，* resetkeys:在键模式列表里面清空所有的键模式，例如ACL ~foo:* ~bar:* resetkeys ~objects:*，将会使用户端只有权限访问objects:* 的键模式。
配置有效的用户名密码：
`><password>:`添加密码到用户有效密码列表里，例如>mypass将添加mypass到用户有效密码列表，这个配置会使nopass失效，每个用户都可以配置任何数量的密码。
 `<<password>:`从用户有效密码列表中删除密码，Redis会返回错误如果这个密码在之前没有被设置的话。
`#<hash>:`添加SHA-256形式哈希值到用户有效密码列表里，这种方式允许用户使用哈希值在acl.conf中存储密码，以避免明文形式存储。只有SHA-256形式哈希值才能被当做密码的哈希值，并且只能是64字符长度，并且使用16进制编码的小写字符。
`!<hash>:`从有效的密码列表中删除哈希值密码，这种方式在你不知道密码明文的情况下却又想删除密码的时候非常有用。
`nopass:`删除所有与用户关联的密码，并且用户会被标记不需要密码验证：这意味着任何密码都会通过用户验证。如果在默认（default）用户使用这个配置，所有客户端连接会立即通过默认（default）用户验证。注意resetpass配置会清除这个配置。
`resetpass:`清除用户所有密码，另外会清除用户nopass状态。使用resetpass后用户没有与其相关联的密码，这种方式下用户如果不添加新密码的话（或在后面设置nopass）则无法进行验证。
注意：未使用nopass进行标记且没有有效密码列表的用户实际上是无法使用的，因为将无法以该用户身份登录。
##重设用户
reset：reset和以下操作等同：resetpass, resetkeys, off, -@all.用户将返回和它被默认创建时同样的状态。
使用ACL SETUSER创建和修改用户当前ACL配置
用户可以以下面两种方式被创建和修改：
1.	使用ACL命令和ACL SETUSER子命令。
2.	修改Redis服务器配置，用户可以在那里被定义，然后重启服务器并生效。 或者使用外部ACL文件，使用ACL LOAD 来导入ACL信息。
在这部分我们会学到怎样使用ACL命令来定义用户。如果我们学会用这种方式定义用户转换为用配置文件配置用户就会很简单。 使用配置文件定义用户会在下面的章节单独列出。
开始我们使用最简单的方式定义一个ACL用户：

`> ACL SETUSER alice`
OK`
SETUSER命令需要提供用户名以及与用户相关联的ACL规则。但是在上面的例子里我们没有提供任何与用户相关联的规则。这样我们只是创建一个用户，如果用户不存在的话，会使用默认ACL配置属性创建。 如果用户已经存在，这条命令将不起任何作用。
让我们来看一下默认用户的状态：
`> ACL LIST
1) "user alice off -@all"
2) "user default on nopass ~* +@all"
`

刚刚创建的alice用户状态是：
Off关闭状态：意思是用户被禁用，AUTH命令将不会起作用。
不能使用任何命令，注意默认创建的用户没有使用任何命令的权限，所以-@all在上面的输出可以忽略，但是ACL LIST会用更显式的方式来告诉用户。
最后用户不能访问任何键模式。
用户没有任何密码设置。

这样的用户是完全没有用的。让我们来定义一个开启的，设置密码的，只能访问GET命令并只能访问cached：键模式的用户。

`> ACL SETUSER alice on >p1pp0 ~cached:* +get`
`OK`

现在用户可以做一些事情，但如果做其他的事情会被拒绝：

`> AUTH alice p1pp0
OK
> GET foo
(error) NOPERM this user has no permissions to access one of the keys used as arguments
> GET cached:1234
(nil)
> SET cached:1234 zap
(error) NOPERM this user has no permissions to run the 'set' command or its subcommand
`
命令像预期想象的一样可以工作，为了检查用户alice的配置属性（记住用户名是区分大小写的），可以使用一个对计算机更友好的替代ACL LIST命令，ACL LIST的输出被认为是对人更友好的。

`> ACL GETUSER alice
1) "flags"
2) 1) "on"
3) "passwords"
4) 1) "2d9c75..."
5) "commands"
6) "-@all +get"
7) "keys"
8) 1) "cached:*"
`

ACL GETUSER命令返回的是更容易用电脑进行提取的键值对。输出包括用户标志，键模式列表，密码列表等。如果我们使用RESP3这种输出会更友好，如果使用RESP3方式，返回值如下：

`> ACL GETUSER alice
1# "flags" => 1~ "on"
2# "passwords" => 1) "2d9c75..."
3# "commands" => "-@all +get"
4# "keys" => 1) "cached:*"
`

注意：从现在开始我们会继续使用Redis默认协议，版本2，因为转换到版本3对于社区和使用者来说还需要一段时间。
使用另一个ACL SETUSER命令（在另一个用户下执行，因为Alice没有权限运行ACL命令）我们可以为用户增加更多的模式：

`> ACL SETUSER alice ~objects:* ~items:* ~public:*
OK
> ACL LIST
1) "user alice on >2d9c75... ~cached:* ~objects:* ~items:* ~public:* -@all +get"
2) "user default on nopass ~* +@all"
`

现在用户的表述和和我们想象的完全一样。

多次调用ACL SETUSER命令时的情况
在这里需要特别理解如果多次调用ACL SETUSER命令时的情况。在多次调ACL SETUSER时，SETUSER命令不会重新设定用户，而是在之前设定的基础上应用新的ACL规则。用户只会在之前不存在的时候被重新设定：在这种情况下，一个新的用户会被创建，没有关联任何ACL规则，换句话说，用户不能做任何事情，是被禁用的，并且没有密码相关联：从安全的角度上来说这种默认方式是最好的。
但是后面的ACL SETUSER调用只会在之前的基础上增加ACL规则，例如下面的例子

`> ACL SETUSER myuser +set
OK
> ACL SETUSER myuser +get
OK
`
会赋予用户调用GET和SET的权限。

`> ACL LIST
1) "user default on nopass ~* +@all"
2) "user myuser off -@all +set +get"
`
使用命令类别：
设置用户ACL规则用逐步添加命令的方式是件很懊恼的事情，所以我们会用下面一种方式：
`> ACL SETUSER antirez on +@all -@dangerous >42a979... ~*`

通过使用+ @ all和-@ dangerous，我们包含了所有命令，然后在Redis命令表中删除了所有标记为危险的命令。请注意，除了+ @ all外，命令类别从不包含模块命令。如果使用+ @ all，则所有命令都可以由用户执行，甚至未来通过Redis模块加载的命令也可以执行。但是，如果使用ACL规则+ @ readonly或其他任何规则，则始终会排除Redis模块命令。这一点非常重要，因为我们只应该信任redis原生的命令。Redis模块有些时候会引起程序执行的风险，并且在ACL仅是+的情况下，即+ @ all -...的形式，我们应该绝对确保可执行命令中不会有危险的命令。
 但是我们要记住要命令类别，并且命令类别中确切包含的命令是不可能的。因此Redis ACL命令导出CAT子命令，该命令可以两种形式使用：

`ACL CAT -- Will just list all the categories available
ACL CAT <category-name> -- Will list all the commands inside the category
`

例子：

` > ACL CAT
 1) "keyspace"
 2) "read"
 3) "write"
 4) "set"
 5) "sortedset"
 6) "list"
 7) "hash"
 8) "string"
 9) "bitmap"
10) "hyperloglog"
11) "geo"
12) "stream"
13) "pubsub"
14) "admin"
15) "fast"
16) "slow"
17) "blocking"
18) "dangerous"
19) "connection"
20) "transaction"
21) "scripting"
`
我们会看到一共有21个类别，下面让我们看一下geo类别中包含的命令：
`> ACL CAT geo
1) "geohash"
2) "georadius_ro"
3) "georadiusbymember"
4) "geopos"
5) "geoadd"
6) "georadiusbymember_ro"
7) "geodist"
8) "georadius"
`
注意命令可能同时属于不同的类别，所以ACL规则例如+@geo -@readonly会导致特定的geo命令被排除，因为它们也属于readonly类别。

增加子命令：
通常来说提供将命令整体加入ACL可执行命令列表或者从列表中删除是不够的，很多Redis命令通过子命令做很多不同的事。例如CLIENT命令可以被用于执行危险或不危险的操作。很多项目部署不会将CLIENT KILL交给非管理员用户执行，但是却允许它们使用CLIENT SETNAME来设置连接用户属性。
注意：新的RESP3协议的 HELLO命令将来会提供一个SETNAME的选项，但是这个还是一个好的例子。
在这种情况下我们可以用下面的方式改变ACL规则：
`ACL SETUSER myuser -client +client|setname +client|getname`
我们先删除CLIENT命令，然后再加上两个可以执行的子命令。注意我们不能改变这个顺序，子命令只可以添加而不能删除， 这样设计的原因是也许将来会有更多的子命令被定义，显式的指定哪些子命令用户可以被执行会安全得多。另外，如果添加子命令的父命令不是被禁止的，会产生错误，因为在这种方式下只可能是用户定义的ACL规则有错误。
`> ACL SETUSER default +debug|segfault
(error) ERR Error in ACL SETUSER modifier '+debug|segfault': Adding a
subcommand of a command already fully added is not allowed. Remove the
command to start. Example: -DEBUG +DEBUG|DIGEST
`
值得注意的是子命令匹配会带来一些服务器性能效率上的问题，但是这类的问题即使使用合成性能测试基准也非常难测量，当这类命令执行的时候只有额外cpu被消耗，其他命令则不会被消耗。

+@all 和 -@all对比

在上一章节我们看到怎样通过添加删除单独命令来定义acl命令规则。

密码在内部是怎样存储的

Redis内部通过SHA256哈希算法存储密码，如果你定义一个密码并且查看ACL LIST 或GETUSER的命令输出，你会看到一个看起来像伪随机的16进制字符串。下面有个例子，因为在之前的例子为了简单起见16进制字符串只截取了前一部分。

`> ACL GETUSER default
1) "flags"
2) 1) "on"
   2) "allkeys"
   3) "allcommands"
3) "passwords"
4) 1) "2d9c75273d72b32df726fb545c8a4edc719f0a95a6fd993950b10c474ad9c927"
5) "commands"
6) "+@all"
7) "keys"
8) 1) "*"`

并且旧命令CONFIG GET requirepass从redis 6版本开始不会返回明文密码，而是会返回加密的密码。
适用SHA256算法可以避免存储明文密码，与此同时，也支持非常快的执行AUTH命令，这也是Redis重要的特点也是客户希望从Redis得到的。

但是ACL密码不是真正的密码，而是客户端和服务器端的共享秘钥。因为这个原因密码不是一个客户使用的验证令牌（authentication token）。 例如：
密码没有长度限制，密码只是存储在客户端的一些软件里，人们不会需要从这里获取密码。
ACL密码不会用于保护其他任何东西，例如永远不会作为一些电子邮箱的密码。
经常的如果你可以访问密码的哈希值，并拥有访问特定服务器权限，或者破坏系统本身，你已经可以访问密码所保护的实体，Redis实例的稳定和它所存储的数据。
因为这个原因去减慢密码验证速度而去使用耗时和耗空间的算法去追求密码的安全性，是一个非常不明智的选择。我们建议的方式是生成一个难以破解的密码，因此即使使用哈希算法也没有人可以使用字典或暴力破解去破解密码。因为这个原因，一个特别的ACL命令使用系统的伪加密生成算法去生成密码。

`> ACL GENPASS
"0e8ad12c1962355a3eb35e0ca686343b"
`

这个命令会输出16字节（128位）随机字符串转换为32字节字母数字字符串。这个长度密码完全可以避免被攻击并且足够短被管理，复制粘贴，存储等等。这个是你生成Redis密码应该被用到的。

使用外部ACL文件	

通过Redis配置方式存储ACL用户有以下两种方式：
1.	用户可以直接在Redis.conf文件中被指定。
2.	可以通过外部ACL文件指定。
这两种方式是互相不兼容的，Redis会让你选择其中的一种方式。在redis.conf中配置用户是非常简单的，适用于简易的应用场景。如果需要有很多用户去设定，在复杂的环境中，我们强烈建议使用ACL文件。
Redis.conf 和外部ACL文件是完全一样的。所以从一个转换到另一个是非常简单的。
下面的例子：
`user <username> ... acl rules ...`
例如：
`user worker +@list +@connection ~jobs:* on >ffa9203c493aa99`
当你希望使用外部ACL文件，你需要指定配置选项aclfile，像下面这样：
`aclfile /etc/redis/users.acl`
当你直接在redis.conf中直接指定一些用户的时候，你可以使用CONFIG REWRITE来覆盖存储文件中新的用户配置。
外部ACL文件具有更加强大的功能，你可以像下面这样：

使用ACL LOAD如果你手动的改变ACL文件并且希望Redis重新读取新的配置，注意命令能够读取文件的前提是所有用户在文件中都被正确设置。不然的话会返回错误，并且旧的配置会继续生效。
适用ACL SAVE来存储当前ACL配置到ACL文件中。
注意CONFIG REWRITE不会自动调用ACL SAVE：当你使用ACL文件的时候配置和ACL是被分开处理的。





















