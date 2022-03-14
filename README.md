简介
------------
OpenPasswordFilter是一个开源的自定义密码过滤器DLL和用户空间服务，以更好地保护/控制活动目录域密码。

这个想法的起源来自于进行了许多渗透测试，在这些测试中，组织有选择普通密码的用户，以及控制这种行为的最终困难。事实是，任何规模的域都会有一些用户选择Password1或Summer2015或Company123作为他们的密码。任何入侵者或低权限用户，如果能够猜到或获得域的用户名，就可以很容易地通过这些非常普通的密码，开始扩大域的访问级别。

微软在活动目录中提供了一个奇妙的功能，那就是能够创建一个自定义的密码过滤器DLL。这个DLL在启动时被LSASS加载（如果已配置），并将被查询到用户试图设置的每个新密码。DLL简单地回复TRUE或FALSE，以表明该密码通过或未通过测试。

有一些商业选择，但它们通常属于 "收费 "类别，这使得一些组织在对这类非常常见的坏密码实施真正有效的预防控制时有些望而却步。

这就是OpenPasswordFilter的用武之地--一个开源的解决方案，可以对常见的密码添加基本的基于字典的拒绝。以及调用haveibeenpwned.com的API进行检查。

OPF由两个主要部分组成。

   1. OpenPasswordFilter.dll -- 这是一个自定义的密码过滤器DLL，可由LSASS加载，以审查传入的密码变化。
   2. OPFService.exe -- 这是一个基于C#的服务二进制文件，提供一个本地用户空间服务，用于维护字典和服务请求。
  
DLL与环回（loopback）网络接口上的服务进行通信，根据配置的数据库检查密码禁止值的数据库来检查密码。pwnedpasswords API(https://haveibeenpwned.com/) ，用于确保账户的SAMAccountName、名字、姓氏surname和显示名不在公开泄露密码中。之所以选择这种架构，是因为启动后很难重新加载DLL，而且管理员很可能不愿意在想把另一个禁止的密码添加到列表中时重启他们的DC。只要记住这个架构是如何工作的，你就会明白发生了什么。

**注意**目前的版本是非常ALPHA的!  我已经在我的一些DC上进行了测试，但你的里程可能会有所不同，你可能希望在现实生活中使用它之前在安全的地方进行测试。


安装指引
------------
你可以下载已经编译完成的X64环境的安装包:

[64位程序](https://github.com//OpenPasswordFilter/tree/master/x64/Release)

你要对DLL进行配置，以便Windows能够加载它来过滤密码。请注意，你必须在所有域控制器上这样做因为任何一个域控制器都可能最终为密码更改请求提供服务。 这里有一个链接到微软的设置密码过滤器的文档的链接。

    https://docs.microsoft.com/zh-cn/windows/win32/secmgmt/installing-and-registering-a-password-filter-dll?redirectedfrom=MSDN
    
上面文档最重要的内容是:

  1. Copy `OpenPasswordFilter.dll` to `%WINDIR%\System32`
  2. Configure the `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Notification Packages` registry key with the DLL name
  
注意，不要在注册表键中包括".dll "扩展名 -- 只需要 "OpenPasswordFilter"这个dll名字。

接下来，要配置OPF服务。 你可以按以下方式进行。

    > sc create OPF binPath= <full path to exe>\opfservice.exe start= boot

然后，你可以从命令行（以管理员身份）启动或停止OPF服务：。
    
    > NET START OPF

or

    > NET STOP OPF

最后，在SYSVOL路径中创建几个字典文件，'\\127.0.0.1\sysvol\testdomain.com（替换你的域名）\OPF\’。这些文件在SYSVOL中，以便它们在所有的域控制器中保持同步，并且在开始服务时检查文件的修改时间，如果它有变化，将再次读入列表，因此重新启动 OPF不需要修改列表时的服务。

These are
- `opfmatch.txt`
- `opfcont.txt`
- `opfregex.txt`
- `opfgroups.txt`

或者你可以跳过这一切，使用安装程序. 

   https://github.com/miumiug/OpenPasswordFilter/blob/master/OPFInstaller_x64.zip

### opfmatch.txt and opfcont.txt
这些应包含每行一个禁止的密码，如：

    Password1
    Password2
    Company123
    Summer15
    Summer2015
    ...

`opfmatch.txt`中的密码将被测试为完全匹配，而`opfcont.txt`中的密码将被测试为部分匹配。这对于拒绝任何包含有毒字符串的密码是很有用的，如`password`和`welcome`。
建议构建一个黑名单的列表，然后使用hashcat规则来建立`opfcont.txt`，其中包含用户可能会尝试的各种字符，像这样。

`hashcat -r /usr/share/hashcat/rules/Incisive-leetspeak.rule --stdout seedwordlist | tr A-Z a-z | sort | uniq > opfcont.txt`

请记住，如果你使用类似unix的系统来创建你的词表，那么行的结束符将需要改变为Windows的格式。

`unix2dos opfcont.txt`

### opfregex.txt
与opfmatch和opfconf文件类似，在这里包括正则表达式--每行一个--用于无效的密码。例如。
包括'xx.*xx'将捕捉所有有两个x的密码，后面是任何有两个x的文本。
保持这个列表简短，因为正则表达式匹配在计算上比简单的匹配或包含搜索要昂贵。

### opfgroups.txt
该文件包含零个或多个Active-Directory组的名称--每行一个。这些可以是安全组或分配组。
如果一个用户是该组的后裔子孙，则被认为是在该组中。一个用户的密码只会被检查 
如果该用户是这个文件中列出的任何组的成员，他的密码才会被检查。如果该文件存在但不包含组，那么每个用户都将被检查。

## Event Logging日志记录
opfservice.exe应用程序使用代码100和101记录到应用程序事件日志。搜索事件日志将确定opfservice正在检查什么。
如果服务无法启动，很可能是摄取词表时出错了，问题条目的行号将被写入应用程序事件日志。

## 其他补充
这需要一个64位的操作系统，因为密码过滤器的位数必须与操作系统的位数一致。

安装程序包含密码黑名单列表。匹配列表是rockyou.txt，每一行少于10个字符的内容都被剥离出来，并排序和去掉冗余。
opfgroups和opfregex列表是空的。

如果一切顺利，重新启动你的DC，并通过使用正常的GUI密码重置功能来测试，选择一个在你的禁止列表中的密码。

