---
title: SteamDeck完全指南-以一名主机玩家视角
date: 2022-12-31 10:56:41
categories: Game
tags:
- Steam
- SteamOS
- Game
---

> 熟悉我的人都知道我其实是一个游戏爱好者，只是很少在博客写非技术文章而已。我在11月左右，因为受不了某日厂的PC独占行为而决定入手Steam Deck，这也是我自从2017年彻底放弃PC阵营之后第一次重回PC游戏领域，因此这里从一个主机玩家的视角整理一下我自己对Steam Deck，SteamOS的一些指南，希望能帮助中文领域的类似玩家快速上手和方便折腾。
> 
> 断断续续写了几个小时，后续不断把我遇到的一些折腾指南都在这里更新吧，可以借助目录树来查看感兴趣的内容

# Steam Deck购买

## 主机

Steam Deck官网：https://www.steamdeck.com/zh-cn/

主机的规格主要是：
1. CPU：AMD APU，4核心Zen2架构
2. GPU：AMD APU，RDNA2，1.6 teraflops
3. RAM：16GB LPDDR5，不可选配
4. 分辨率：1280x800
5. 电池：40WH
6. 重量：669克

主机款式目前分为以下三种：

1. 64GB eMMC：$399，推荐自购SSD更换
2. 256GB SSD：$529，不推荐
3. 512GB SSD：$649，防眩光屏 + 一堆虚拟物品，不差钱用

大众的选择一般是64GB款 + 自购512G的SSD（M\.2 2230)更换，更换教程也[全网都有](https://www.ifixit.com/Guide/Steam+Deck+SSD+Replacement/148989)不麻烦（一把十字螺丝刀可搞定）。

64GB款国内一般直接TB代购在¥3100，自己直接走美区充值余额 + 海淘关税13% + 国内转运，成本在¥2900左右，就算再加上JD的512GB的SSD ¥500价格，最贵也就¥3600，实际算下来是很香的。

虽然这样说，实际上我当时为了尽快上手且为了图省事，一步到位购买了现货顶配版（¥5000），防炫光玻璃效果也是有的，肉疼就肉疼吧，当是早买早享受了：）

![1](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-12-31/images/1.jpg)


到手后拿起来比Switch重很多，实际体验下来并不适合长期举着玩，除了放在底座上手柄来玩大作外，推荐的掌机玩法要么架在桌子/腿上玩，要么直接侧躺靠着握把玩（主要是文字类型游戏，注意视力）

## 存储卡

Steam Deck支持TF存储卡扩展，我这里选择512GB以后自然没必要换SSD了，但是为了后续可能用到的空间，以及安装Windows到TF卡上，所以又300¥买了一个UH3的512G 闪迪TF卡，直接在机身下方插入即可。

默认SteamOS会推荐格式化为EXFAT文件系统，并且可以选择默认安装游戏到TF卡上，和Switch的逻辑有点像。

但是我为了后续安装Windows，最终又制作成了Win To Go，将TF卡格式化为NTFS文件系统之后，SteamOS下就不再显示改安装路径了……这点暂时还没管，应该后续可以通过插件解决（折腾预定）

## 配件

我作为主机玩家，且主要目的是为了玩日厂的JRPG，因此连接电视+手柄对我来说是必不可少的。

Steam Deck官方提供了一个基座（扩展坞）：https://www.steamdeck.com/zh-cn/dock

接口规格为：
1. 3个USB-A 3\.1接口
2. 1个HDMI 2\.0接口
3. 1个DP 1\.4接口
4. 1个供电接口
5. 1个RJ45网口

值得注意的是，不同于Switch的基座，这个基座只提供USB Type-C的扩展能力和充电能力，完全不能提升游戏性能（实测，对比直接Type-C供电，游戏帧率一致）。

显示器输出虽然说最高支持4K 120Hz，实际上游戏压根带不动。推荐的游戏输出分辨率是1080P（后文提），另外还有一个注意点，官方基座MST（多显示器输出）需要同时接入HDMI和DP两个端口而不能二次转接，不过实际性能表现带单显示器已经极限，大部分人压根用不到

官方基座售价$89，国内现货800¥起步，完全不值得购买（不差钱另说），其实如果你不追求长期接电视/显示器，选择一个便宜的100¥以内的Type-C转HDMI头都可以解决

我最终选择了一个第三方的基座，除了没有DP接口其他规格完全一致，只需要¥250，铝合金质感比官方基座的塑料明显要好（就离谱）

![2](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-12-31/images/2.jpg)

# Steam账号和支付

自从2017年彻底退出PC游戏之后，我的国区Steam账号终于又活过来了。(上次登录1400天前)

Steam账号体系和PS的账号体系更类似（对比Switch和Xbox那种随时跨区切换购买游戏而言），一个账号每3个月才能更改一次地区，且余额会按照汇率等价兑换。另外注意转区完成以后还需要进行一次有效购买才可以实际切换

但是，Steam对转区判定比较严格，需要挂对应地区IP代理，否则经常性转换可能被红信（即Steam的欺诈警告，会锁定账号的游戏购买）

由于不同区的游戏价格差异很大，可以参考[SteamDB](https://steamdb.info/)这里搜索对比一下，且有些游戏会锁国区（暴力血腥or小黄油），推荐的方式是类PS的账号体系的应对方式，我们注册多个地区的不同账号，通过在一台Steam Deck上登陆多账号，然后切换游玩（还可以利用家庭共享来让账号1游玩账号2的游戏，并成就和存档挂在账号1，老PS玩家很熟悉）

## 注册Steam外区账号

注册Steam外区账号需要使用对应地区的IP代理，这个随便找个比如[Free Proxy](http://free-proxy.cz/)，或者利用一些加速器自带的商店加速能力就可以切。

推荐注册的外区是阿根廷（大部分游戏的最低价地区，但锁本地信用卡和充值卡，后文讲），或者土耳其（目前截止12月，土耳其支持国内Visa/Mastercard双币卡，非常方便）

注册完成以后，需要进行一次商店定区，无论是外区货币充值还是使用占位符兑换码兑换一次都可以，保证商店页面显示的货币为外币即可。

另外，强烈建议注册后立即下载Steam手机版，登陆外区并绑定手机两步认证（不需要外区手机号，国区即可接），为后面的Steam市场开放和余额购买做准备

## 支付和余额购买

土耳其区因为暂时支持国内双币卡，就不多说了，这年头主机玩家没个Visa/Mastercard信用卡不太可能。注意账单地址选择一个真实的土耳其地址即可。当然你也可以选择下面提到的余额购买方式，也可以直接选择电子充值卡兑换码（一般和汇率持平）

阿根廷区因为最低价被大量玩家滥用，既锁本地信用卡，又在2021年关闭了电子充值卡的购买手段，目前我已知支付手段包括：

1. 日韩国等地的电话亭余额充值：新号可以充，一般需要联系下当地认识的人，或者TB代充，目前我见到的汇率约为1000比索:48人民币，属于小亏级别，不推荐长期使用
2. 当地信用卡代充：新号可以充，阿根廷区比较少见，原因是大量多账号用一个信用卡可能被Steam风控
3. Steam市场购买余额：需要开通手机两步认证，且购买等价5$游戏不退款满7天后即可开放。利用Steam市场可以卖虚拟物品，自动转换汇率的特点，一般叫做饰品交易，俗称“挂刀”。本质上原因是Steam余额无法直接提现，因此会有人愿意在外部平台，以低价现金卖出虚拟物品，然后你以低价买到，再拿到市场高价卖出得到账户余额（Steam市场还会扣交易税15%）。这个TB也挺多的，目前我见到的汇率约为1000比索:38人民币，属于小赚水平

![5](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-12-31/images/5.png)



# SteamOS使用

SteamOS的体验，简而言之可以说是把PC上的Steam客户端，做到了操作系统级别的体验，基本交互比PC上的大屏幕模式更舒服，且所有导航设置都适配了手柄（无论是主机手柄还是外接PS/Xbox手柄甚至是Switch手柄）

SteamOS基于[Arch Linux](https://wiki.archlinux.org/title/Steam_Deck)，提供了游戏模式（大部分时间在这里）以及桌面模式。游戏模式自带了一个快捷菜单键，类似PS键，除了可以进行除了WiFi/蓝牙/飞行/亮度等调节，查看通知邀请啥的常见能力，最有意思的是可以进行性能配置，包括限制电量TDP来控制续航（只有40WH的电量，意味着功耗拉满25W，只能支撑1个半小时），锁帧率，以及开启采样技术等，后文提。

桌面模式是[KDE](https://wiki.archlinux.org/title/KDE)，我第一次上手感觉和macOS的不太像，更像是Windows桌面的逻辑，不是很舒服，熟悉一段之后还好。文件管理器叫做[Dolphin](https://userbase.kde.org/Dolphin/File_Management)，使用起来反而更像macOS的Finder，标签页，边栏，打开终端啥的。终端模拟器叫做[Konsole](https://konsole.kde.org/)，我说实话比macOS的终端反而还好用……桌面模式的Steam客户端可以进行一些复杂操作（实际上桌面模式上的Steam客户端和PC上操作完全一样），如添加非Steam游戏快捷方式，后文提。

对于Windows游戏来说，SteamOS内置的[Proton兼容层](https://github.com/ValveSoftware/Proton)提供了转译。这个转译是API级别的（即实现了一套Win32 API，\.NET API，以及DirectX转译Vulkan等），不是类似Apple M1对x86_64的指令集转译，在我测试游戏中表现挺好的，兼容性不错，帧率甚至超越Windows原生执行。

## 游戏外接显示器分辨率

SteamOS和桌面模式默认情况下连接显示器后，就能以显示器原生分辨率（我测试过1080P和4K分辨率）显示UI，但是这和游戏分辨率是两回事。

当启动游戏后，Steam默认配置会把游戏锁定在1280x800，很多游戏感到非常的模糊（比Switch接电视还低）。查了一下才发现需要每个游戏进行配置，选中`游戏` -> `属性...` -> `游戏分辨率`，从"Default"改为"Native"（指的是显示器原生分辨率）

这点想吐槽的是竟然没有全局开关，也许是为了适配各种PC游戏配置要求和支持分辨率混乱的现状……不过输出1080P甚至4K之后，明显感觉Steam Deck性能吃紧，大部分近两年的3D游戏都很难30帧以上运行，只有2D游戏可以继续拉满。这里就要提到下面的性能配置，能利用超采样技术以达到分辨率和性能兼得。

## 性能配置和全局FSR

SteamOS的快捷菜单能进行各项性能配置，除了能有一个专门的浮层显示帧数和APU/RAM占用率等指标外，并且还能选择是全局配置还是仅当前游戏配置，非常灵活。看了下包括：

1. 锁帧（一般默认60即可）
2. 半速率着色（[VRS](https://learn.microsoft.com/zh-cn/windows/win32/direct3d12/vrs)）：可以省电，按照其解释会稍微影响输出分辨率，但是我测试几个游戏后没看出明显差异（大作掌机模式必开）
3. TDP限制：功耗限制，掌机模式必限制10W否则马上没电
4. GPU锁频：可以追求极限帧率，供电模式下拉满，疑似怀疑长期可能会影响GPU寿命
5. 缩放过滤器：低分辩率超采样到高分辨率，只推荐开[FSR](https://www.amd.com/en/technologies/fidelityfx-super-resolution)，其他采样效果一般

![3](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-12-31/images/3.jpg)


其中最有用的是这个全局FSR，Steam Deck的APU本身性能带一些CPU瓶颈的大作（如老头环），模拟器游戏等，明显会感到吃力，供电下只有30帧出头，基本属于不能玩水平（同期Switch/PS4：30帧流畅游玩）

不同于Windows下这种技术，需要每个游戏厂商自己单独支持，并在游戏内单独开启，这种全局FSR依赖了DirectX到Vulkan转译层直接全局干上，效果非常明显。

我用电视体验和实践下来，基本养成了习惯，对3D大作，先在Steam属性里改为输出1080P，然后游戏内设置720P的分辨率，最后菜单仅当前游戏配置FSR，这样3D模型会采样到1080P，同时帧率可以到40-60畅玩，只有2D纹理的文字会模糊点，但是体验明显会爽很多。

有一个注意的点，大部分游戏都需要在游戏内选择窗口模式而不是全屏模式，才能正常FSR生效（打开性能面板查看有一行"FSR：ON"）。虽然实际上游戏模式下，压根没有"窗口模式"全部都给你转译拉成全屏了，只有桌面模式才能窗口，这一点不知道是不是Bug？

总之基本所有3D游戏进去就找窗口模式+FSR就行，2D游戏发现有时候窗口模式会导致文字变模糊，所以不要动。总之PC游戏就是一个“配置灵活”（折腾）

## Steam控制器映射

PC游戏，众所周知就是优先按照键鼠交互开发的，大部分游戏都没有一方对手柄进行支持（大作或者主机移植的游戏基本才有）。因此Steam Deck提供了我见到主机最多的手柄按键，以及每个游戏级别的映射布局。

进入游戏后按Steam键就能看到“控制器设置”，里面可以看到每一个控制器（比如我电视玩的时候用PS5手柄，就能看到两排）的布局。这里可以选择基本就是：

1. 键鼠（基本就是右摇杆鼠标，L1左键 R2右键，左摇杆对应WASD，触摸板也是鼠标）
2. 手柄（指的是游戏开发利用了[SteamInput](https://steaminput.wiki/en/intro/steam-input)，XInput，DirectInput等原生控制器支持）
3. 支持视角控制的手柄（区别是右触摸板可以控制精确视角，FPS游戏多，需要游戏支持）

当然，这里还可以切换到社区布局，下载其他人的布局映射文件，热门游戏都有很好的布局文件。另外这个配置会和你的Steam账户云保存起来，用PC加的配置也能用跨机器用），挺好的

另外，非Steam游戏（无论是通过Proton转译层的游戏，还是Chrome和Dolphin这种原生Linux），也可以进行映射，甚至能根据游戏名（可以改，下提）共享社区映射，这也是另一个我认为SteamOS比Windows好用的地方。

## 添加非Steam游戏

我们可以把桌面模式的应用添加到Steam吗？当然可以，比如SteamOS就会引导你添加Chrome到Steam里，这下真成了iPad之外的便携浏览器了。你可以添加各种工具，甚至VSCode，也有专门人分享的布局文件方便手柄操作😂

以及，在有时候我们不得已要用非Steam的Windows程序，如游戏启动器、汉化补丁、“学习版游戏”：），这时候我们肯定要Proton转译层，从命令行直接调用非常复杂我也懒得去看，利用把exe可执行程序添加到Steam库中，我们就能直接选择Proton转译并且享受各种便利（包括手柄映射，分辨率输出调整，截图管理，Shadercache等）

按照[官方说明](https://help.steampowered.com/zh-cn/faqs/view/4b8b-9697-2338-40ec)，添加方式需要进入到桌面模式（不理解为啥游戏模式没有入口），右键库选择“添加非Steam游戏到我的库中”，默认会显示桌面模式安装的原生Linux应用列表，我们不管，选择新增路径。

此时弹出的文件管理器中，可以选择不同硬盘的程序，比如机身里的（`/home/deck/`下），TF卡里的（`/run/media/deck/TF卡序号/`)，无论是EXFAT还是NTFS文件系统的都能添加（用这个可以实现SteamOS和Windows双系统共享一个游戏，进度靠Steam云存档）。注意下方扩展名类型要选为"All"不然无法显示exe可执行程序

路径选择完成后再点一次“添加所选程序”就可以在库里看到了，默认游戏名是exe可执行程序文件名，右键`属性`可以改名一下，然后选择兼容层选择最新版本的Proton（还有一个Proton Experimental不过我没用过），双击测试执行效果即可。

## Proton兼容层

Proton兼容层是基于Wine的改进项目，虽然开源且在任意Linux上可用，但是Steam Deck用的版本有Value内部定制，对游戏的兼容性，以及执行速度有着非常明显的优势（以后就不用Wine了）😂

其中，Steam Deck有自己的一个[Deck Verified](https://www.steamdeck.com/verified)标签，分为四档，其实一般来说除了不支持的档位以外都可以认为是可玩的。

另外，Deck Verified是有时效性的，有些游戏可能最新的SteamOS更新后就能正常运行，但是依然显示不支持，这种情况可以借助社区提供的：[ProtonDB](https://www.protondb.com/)网页，查看其他玩家上传的实际体验（不过有些可能不是Steam Deck用户而是Linux PC+开源Proton用户，不可全信）

当然，实际上我买游戏时也不怎么看Deck Verified，实际能不能跑下载下来测一下便知（目前我库里不能跑的是2/80，极少），反正Steam不像主机厂商，[游戏2小时内可以无条件退款](https://store.steampowered.com/steam_refunds/?l=schinese)，因此自己测试自己的库里的游戏时最可靠的。


## 疑难解答

### 定期清理Shadercache/Compatdata

这两个是SteamOS的存储容量“其他”的罪魁祸首。

Steam游戏默认配置会开启Shadercache，因为Steam Deck硬件配置的唯一性，基本你安装所有的游戏，都会提前下载好离线编译好的Shader，不再需要运行时编译，大大减少游戏第一次加载和场景卡顿。甚至非Steam游戏也会把转移时编译的Shader缓存起来，这项功能是全局生效的（不同于Windows需要游戏厂商支持）

Shadercache路径在`/home/deck/.local/share/Steam/steamapps/shadercache`下（这是Steam客户端根路径）

而Compatdata，是Proton游戏兼容层产生的文件夹，又称pfx，其本质是，Proton因为是模拟Windows的运行环境，其背后会做一个精简的沙盒（纯净大小约为190MB），分配给这个游戏，这个游戏对Windows系统的所有修改都只在这个沙盒中生效，包括注册表，存档文件，甚至可能是黑客破坏（非常Nice）。而我们游玩过一个Windows游戏，而它又被我们从Steam库里删除时候，神奇的是这个Compatdata竟然不会自动删除（Bug？Feature？）

对这两个文件，我找到了一个作者写的好用的工具[Steam Deck: Shader Cache Killer](https://github.com/scawp/Steam-Deck.Shader-Cache-Killer)，使用也很简单，按照说明下载好以后，打开就能看到所有的Shadercache目录和对应游戏的名称，AppID信息，可筛选Non-steam和Uninstalled。

值得注意的是，目前这个工具对非Steam游戏的名称识别并不好，必须你最近启动过这个游戏一次才可以在列表显示（看代码是通过读了`$STEAM/logs/content_log.txt`日志解析的，但是这个日志会定时清理……）。原因是非Steam游戏的AppID是根据“游戏名”+“路径名”的[哈希得到](https://gaming.stackexchange.com/questions/386882/how-do-i-find-the-appid-for-a-non-steam-game-on-steam)，所以不能反推出原游戏名和路径名。

为了防止错误删除了正在玩的沙盒（包括游戏存档），简单傻瓜做法就是定期清理，记录已安装的这些非Steam游戏的AppID（或者无脑就是每次执行清理前先手动启动一次后再清理），每次只保留已添加的非Steam游戏Shadercache和Compatdata目录，其他的一律删除。

### 修改游戏启动路径以及自定义启动参数

Steam商店的游戏都有一个专门的元信息，其可以在[SteamDB](https://steamdb.info/)的Information和Configuration下看到，包括

1. 显示的游戏名
2. 发行商信息
3. 发行日期
4. 启动入口（如启动的exe可执行程序路径，默认参数是什么）

其中我最近就遇到了一个问题，是这个游戏有一个自己的启动器（可能是C#写的），但是Steam游戏模式下，启动器无法弹出而卡住，只有桌面模式能弹出。我想跳过直接执行另一个实际游戏的exe可执行程序（与启动器搏斗和折腾……）

找了一圈，改名符号链接在Proton下也有兼容问题，最终还是简单粗暴，利用这个[Steam Metadata Editor](https://github.com/tralph3/Steam-Metadata-Editor)，可以直接修改库里面游戏的启动入口，包括启动的路径，默认参数等。

此外，它还可以添加多个启动入口，比如有些游戏包括类似游戏本体，启动器，创意工坊Mod编辑器，DLC章节啥的，可以添加不同的入口，方便管理，也不用自己手动对每个exe改成非Steam游戏添加入库。

注意这个工具建议在桌面模式用，不要在Steam客户端启动时修改保存（提前右键退出），才可以生效。

# 安装Windows

前面说了那么多都是SteamOS的用法，但是毕竟Steam Deck本质是一台Portable PC，那么它当然可以安装Windows（官方声明支持Windows 10/11）

虽然Windows系统我从大学就没再用过了（基本只有Mac+主机），但是按照网上的教程摸索差不多实现了从TF卡启动Windows，可以用来逼不得已的情况下跑一些只能在Windows上的软件（Proton兼容失效或者性能有异常的情况）

## Win To Go制作

[Win To Go](https://learn.microsoft.com/en-us/windows/deployment/planning/windows-to-go-overview)制作工具，Windows自带的工具限制必须是“硬盘而非可移动存储设备”才行，比较麻烦还得用DiskGenius改分区，我参考教程用了这个[WTG辅助工具](https://bbs.luobotou.org/thread-761-1-1.html)，下载好Windows 11的镜像，一键写入到TF卡中

## 从TF卡启动并安装Windows

在写好Win To Go到TF卡之后，插入TF卡关机，然后同时长按音量减键和电源键，听到响声后放手，就会进入到启动选项页面。

在Boot Manager中选择"EFI SD/MMD Card"，然后就会进入Windows安装的引导流程了，后面就按照网上常见Windows安装流程走。注意家庭版是需要微软账号和联网的，默认Windows 11有网卡驱动，Windows 10没有，所以建议用专业版😂

安装完成后，会发现默认不再进入SteamOS了，继续进入启动选项页面，这里会出现一个"Windows Boot Manager"在首位，实际上我们Win To Go选择TF卡依旧也能启动，和他没啥关系，可以删除（参考后文“如何默认启动SteamOS而非Windows”）。

![4](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-12-31/images/4.jpg)

## Windows驱动

Steam官方提供了Windows的驱动，可以直接下载好并拷贝到Windows中安装，参考：[Steam Deck - Windows Resources](https://help.steampowered.com/zh-cn/faqs/view/6121-eccd-d643-baa8)

Windows 11上，对inf格式驱动，需要右键并选择“显示更多选项”以查看“安装”选项。

## 疑难解答

### 如何默认启动SteamOS而非Windows

安装Windows之后默认不再会进入SteamOS了，而SteamOS也没提供默认启动选项的能力（这个EFI引导做得比较垃圾），并且一个神奇的Bug，导致无论怎么改默认值，只要启动过一次Windows系统，"Windows Boot Manager"就会默认跑到第一位导致下一次永远默认Windows

我选择的方式是，直接通过禁用"Windows Boot Manager"这个EFI启动项，让SteamOS默认启动，如有需要，长按进入并选择"EFI SD/MMD Card"以启动Windows

具体修改方式可以在SteamOS也可以在Windows下操作，本质都是修改EFI的配置信息，以SteamOS举例子：

1. 打开Konsole
2. 输入`efibootmgr`，会列举查看到对应每个启动选项的数字编号，如`Boot0004 *Windows Boot Manager`
3. 输入`sudo efibootmgr -b 0004 -A`，注意如果没有设置Root密码需要提前用`passwd`设置一次
4. 再次输入`efibootmgr`，此时会不再显示代表激活状态的`*`，如`Boot0004 Windows Boot Manager`

重启可以验证一下，启动列表不再显示这个即可证明成功，默认进SteamOS或者选择`EFI SD/MMD Card`进入Windows

# 总结

其实感觉对于主机玩家来说，Steam Deck的一大缺点，同时也是一大优点就是“可折腾”（Hackable），你总有一种方式，能在掌上玩到PC上的游戏。而对于主机本身配置就很少。Steam Deck它在我看来和Switch压根不是一个竞争对手，而更像是互补。

我能拿着Steam Deck去把打折时买的Roguelike游戏和文字游戏打通，插在电视上跑着40帧带有创意工坊Mod的AAA大作（比如FF7 RE笑），但是绝对不会在Switch上高价买这些PC键鼠设计且没有社区和创意工坊的游戏。

它更像是一个夹在PS5这种沉浸式沙发体验，和Switch掌机轻度娱乐和聚会游戏之间的设备，并且对纯Mac党（没有任何其他Windows设备）是一个非常好的替代品。

就这么多吧，这篇文章大部分都是介绍自己实际遇到的一些问题和教程，希望能对有心入Steam Deck或者遇到类似场景的人提供一些帮助吧：）
