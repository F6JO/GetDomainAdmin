# 获取域控权限的几种思路

对于入侵者来说，进入企业后，为了获取最大权限，通常会将目标瞄准在域控上。下面是针对不同的域控环境，列举的几种不同的攻击攻击手法。

## 一、通过域控相关的漏洞

此思路主要针对域控补丁没有打完整，导致能直接进行域提权漏洞攻击。一般出现在一些安全情报偏弱、补丁更新不频繁的中小公司或者大公司的测试环境。主要是通过MS14068、zerologon、nopac等相关漏洞来直接攻击域控，或者GPP这种能直接获取域管密码的漏洞来直接获取域控权限。

MS14068：https://www.anquanke.com/post/id/172900

NoPAC漏洞利用：https://exploit.ph/cve-2021-42287-cve-2021-42278-weaponisation.html

NoPAC漏洞检测：https://www.trustedsec.com/blog/an-attack-path-mapping-approach-to-cves-2021-42287-and-2021-42278/

NoPAC检测工具的github地址：https://github.com/knightswd/NoPacScan

zerologon漏洞相关：https://blog.zsec.uk/zerologon-attacking-defending/

GPP漏洞：https://cloud.tencent.com/developer/article/1842866

## 二、通过域内的中继

此思路主要针对域控中存在强制NTLM认证漏洞的情况（打印机回调函数、petitpotam等相关漏洞，此漏洞也是potato系列提权的核心。之前微软回应不对此漏洞进行修复，但是在今年4月的补丁中，对相关漏洞进行了修复）。去年爆出的petitpotam漏洞中，通过MS-EFSRPC的越权回调，强制域控和CA进行认证，并在其中做ntlm中继，实现对域控权限的获取。

ADCS relay：https://pentestlab.blog/2021/09/14/petitpotam-ntlm-relay-to-ad-cs/


## 三、通过抓取域管登陆服务器的hash

此思路主要是针对系统运维人员没有对域管账户登陆的服务器进行收敛，随意在服务器上使用高权限账户，并不清理服务器上缓存的hash，导致攻击者能直接在其他服务器中获取到高权限账户的hash，从而实现域内提权，获取域控制器权限。该方法主要的攻击思路就是PTH，通过横向移动+密码抓取，不断获取服务器权限，再在服务器中抓取hash，用新的hash继续PTH获取其他服务器权限，直到抓取到高权限hash为止。


## 四、通过运维人员不恰当的密码管理

在企业中，为了各个部门之间的知识共享和跨部门进行团队协作，需要将一些系统的帐号密码放在统一平台中，方便各个部门使用，此流程多数是通过公司的wiki平台实现。所以有可能能通过在wiki平台搜索相关密码的方式获取到域控密码。除了这种密码随处存放的问题，运维人员还有可能为了方便密码记忆，在高权限账号中使用弱口令，或者通过其他密码能猜测到高权限密码，从而获取到域控权限。

*****

上面几种方法主要是存在中小公司或者大公司的测试环境等安全能力和安全意识偏弱的目标。在大型企业中，域控能自动化打补丁，无法直接通过1day漏洞直接攻击域控，有经验的运维人员也会对高权限账户登陆的服务器进行收敛，直接攻击域控难度过高，于是将目标改为与域控强相关的系统、人员、账户。下面几种是与域控强相关的攻击目标。

## 五、通过攻击Exchange服务器、DNS服务器、SCCM服务器、WSUS服务器等

与域控强相关的系统中，最容易想到的就是Exchange服务器。Exchange服务器具有域控的DCSync权限，通过Exchange的RCE控制Exchange后，也可以直接dump域控中域管的hash，实现对域控的控制。如果获取不到Exchange权限，也可以通过Exchange http的NTLM中继到域控实现域内权限提升（PrivExchange，但是目前已经修复）。DNS服务器中的DNS admin用户也能让域控远程加载自定义的dll文件，从而实现对域控的控制。在SCCM和WSUS服务器中，如果获取到域管hash，则可直接登录域控，如果hash无法登录域控，WSUS服务器则可以直接向域控下发恶意补丁来获取域控权限，SCCM服务器则可以向域控下发恶意应用程序来获取权限。

PrivExchange原理：https://dirkjanm.io/abusing-exchange-one-api-call-away-from-domain-admin/

PrivExchange的github地址：https://github.com/dirkjanm/PrivExchange

DNSAdmin到域控：https://adsecurity.org/?p=4064

WSUS到域控：https://labs.nettitude.com/blog/introducing-sharpwsus/

SCCM到域控：https://labs.nettitude.com/blog/introducing-malsccm/



## 六、通过获取域控制器的localgroup中特权组成员的权限来获取域控权限

企业为了保证域控中的最小权限原则，通常会在域控制器的localgroup中授予一些域用户特殊权限，方便不同人员使用域中的不同能力。比如localgroup中的backup operators组中的成员能对域控进行备份，也就能直接导出域控的SAM数据库，从而获取域管hash。

Backup Operators组成员权限到域控权限：https://github.com/mpgn/BackupOperatorToDA


## 七、通过委派来获取域控权限

在windows域中，有一些服务，需要在其他服务器中执行用户的一些权限，此服务器中的服务账户就会被设置为委派。而如果服务账户被设置为非约束委派，则可通过此服务账户直接获取到域控的权限。

委派相关攻击：https://shu1l.github.io/2020/08/05/kerberos-yu-wei-pai-gong-ji-xue-xi/

## 八、通过域控运维堡垒机

通常在大型企业中，生产网和办公网直接都有隔离，运维人员需要通过堡垒机来对生产网的服务器进行管理。当域控运维人员的堡垒机和其他用户的堡垒机隔离偏弱时，攻击人员也可以通过堡垒机之间的横向移动来控制域控运维人员的堡垒机，从而实现对域控的控制。

## 九、通过运维人员的个人主机

域控运维人员，登陆域控的整个过程中，除了登陆的堡垒机设备外，还需要自己在办公网的办公机。通过内网鱼叉攻击获取横向移动手法，控制域控运维人员的办公机，并在办公机上进行keylogger和屏幕截图，来获取域控相关的权限。

## 十、通过与域控相关的web服务器

在大型企业中，企业运维人员，为了方便域控及其他重要服务器的管理和将自己的能力对公司内部其他部门提供相关服务，需要通过一个web平台来使用域控中的一些功能，或对域控进行管理。因此，会在web中配置与域控相关的信息，会直接与域控进行连接，可以通过在这种web系统中的一些越权或者RCE漏洞来实现对域控的控制。

此攻击思路可以参考ateam的从xxe到域控：https://blog.ateam.qianxin.com/post/zhe-shi-yi-pian-bu-yi-yang-de-zhen-shi-shen-tou-ce-shi-an-li-fen-xi-wen-zhang/#4-xxe-to-%E5%9F%9F%E6%8E%A7



