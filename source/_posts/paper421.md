---
title: 更新RTKLIB 配置文件指南
date: 2019-04-21 19:25:37
tags: 
- rtklib
categories:
- 翻译
 - rtklibexplorer
---



*说明*

> 原文来自于rtklibexplorer的博客文章《[Updated guide to the RTKLIB configuration file](https://rtklibexplorer.wordpress.com/2018/11/27/updated-guide-to-the-rtklib-configuration-file/)》,首先用谷歌翻译进行粗翻译，然后对个别词句做了修改，部分地方做了意译。翻译仅仅为个人阅读方便，有错误的地方还请指出。
>
<!--more-->

[TOC]

自从我上一次更新RTKLIB配置文件指南到现在已经有一段时间了。自上次更新以来，我在代码中又添加了一些新功能，并了解了很多现有的功能。之前更新的内容我都发布在了原帖子里，但这次我想我会重新发布它以便更容易找到它。

RTKLIB的一个好处是它具有极高的可配置性，并且可以提供大量输入选项。但对于使用RTKLIB的新手来说可能会有点压力。[RTKLIB manual](http://www.rtklib.com/prog/manual_2.4.2.pdf)简要解释了每个选项是做什么的，但即使有了它，也很难知道如何最好地为某些参数选值。

我不会在这里对所有的输入选项进行全面的解释，但会解释一些我发现在我的实验中调整有用的内容，和为什么选择这些值。我描述的是一些出现在配置文件中的参数项，而不是出现在RTKNAVI GUI菜单中的选项值，但我为这些参数所做的说明适用于两者。下面的选项里，我把我的最新配置文件与默认配置文件进行了比较并对取值不同的选项进行了说明。下表是流动站5HZ采用率对应的配置文件中的取值。相同的配置文件可用在RTKNAVI，RTKPOST或RNX2RTKP。

以下用黑体(原文为蓝色，译者注）突出显示的设置和选项仅仅在这里的演示代码中提供，而不在发布代码中提供，但我在它们下方给出大部分描述内容则适用于任一代码。我所做的大多数工作都是针对使用Ublox M8N和M8T接收机以及短基线下的RTK解决方案，这些设置多直接应用于这些组合，但作为其他场景的起点应该是应该有用的。
本文主要是对rtklib手册做一些有用的补充，所以对于本文未涉及的一些参数请查找原手册。

## SETTING1：

### pos1-posmode = static, kinematic, **static-start**, movingbase, fixed

如果流动站处于静止状态，请使用“static”模式。如果它正在运动，请使用用“kinematic”或“static-start”。“static-start”模式假设流动站在达到第一次固定之前都是静止的，第一次固定后切换到动态模式，使得卡尔曼滤波能充分利用流动站刚开始时是不运动的这一已知条件。如果基站像流动站一样也在运动，则可以使用“movingbase”模式，但除非基站运动了一段很长的距离，否则不需要用它。我经常发现，即使流动站在运动，“kinematic”模式也能得到比“movingbase”更好的结果。“Movingbase”模式与动态模式并不兼容，所以注意一下不要同时启用这两种模式。如果基站和流动站一直保持固定距离，则在“movingbase”模式下注意设置参数“pos2-baselen”和“pos2-basesig”。如果你知道流动站的确切位置，只想分析一下残差那么可以选择使用`fixed`模式。

### **pos1-frequency = l1**

`L1`对应单频接收机，`L1 + L2`对应双频GPS / GLONASS / Bediou接收机，如果数据中包含Galileo，则选择`L1 + L2 + E5b`.

### pos1-soltype =forward, backward, combined

这是使用卡尔曼滤波的滤波方向。实时处理时，`forward`是你唯一的选择。对于后处理来说，`combined`会先做一次向前滤波，再做一次向后滤波然后对结果进行组合。对一个历元来说，如果该历元两个方向上的滤波都对应固定解，那么组合的结果对应的解状态也是固定的，值取这两个固定解的均值，除非这两者之间的差异太大，在这种情况下解状态将是浮点解。如果只有一个方向有固定解，就取这个固定解的值，状态为固定解。如果两个方向都是浮动解，就取他们的平均值并且状态将是浮动解。`combined`模式并不总是更好的，因为任一方向上的错误固定通常会导致组合结果是浮点解且不正确。`combined`的主要优点是它通常会在数据开头就提供固定解，而单使用仅`forward `模式则需要一段时间才能收敛。2.4.3中的代码在开始做`backwards`之前会重置偏置状态（bias states）以确保解的独立。demo5的代码则不会重置偏置状态，以避免将模糊度的解算方法设置为`continuous`的情况下， having to lock back up when the rover is moving。如果将模糊度的解算方式设置为`fix-and-hold`则需要重置状态偏差。只有当我难以获得初始固定解，但又想知道正确的卫星相位偏差值是多少的时候，我才会用使用`backward`模式做调试。（未翻译好）

### pos1-elmask = 15(degrees)

用于计算位置的最小卫星高程。我通常将其设置为10-15度以减少将多路径引入解决方案的机会，但此设置将取决于流动站环境。天空视图越开放，此值可以设置为越低。

### pos1-snrmask-r = off，pos1-snrmask-b = off，on

用于计算位置的流动站（_r）和基站（_b）的最小卫星SNR。消除不良卫星可能是一个更有效的标准，因为它是一种更直接的信号质量测量方法，但最佳值会随着接收器类型和天线类型而变化，因此我大部分时间都将其关闭，以避免需要对其进行调整为每个应用程序。

### pos1-snrmask_L1 = 35,35,35,35,35,35,35,35,35

设置每五度仰角的SNR阈值。我通常会将所有值保持不变，并根据标称SNR选择35到38分贝之间的值。仅当pos1-snrmask_x设置为on时才使用这些值。如果您使用双频，则还需要设置“pos1-snrmask_L2”

### pos1-dynamics = on

设置动态模式为on时，速度和加速状态将会添加到流动站的卡尔曼滤波估计中去。选择它会改善“kinematic”和“static-stat”的结果，但对“static”模式几乎没有影响。在启用dynamics的情况下，代码的运行速度会明显变慢，但demo5版的代码应该没问题。务必根据流动站的运动特征为流动站设置一个合适的“prnaccelh”和“prnaccelv”。dymamics模式与“movingbase”模式不兼容，因此在使用movingbase模式时要把danamics关掉。

### pos1-posopt1 = off，on（Sat PVC）

设置是否对卫星天线相位中心进行改正。RTK模式下可关闭它，但做PPP的时候需要打开。如果设置为on时则需要在files参数中指定卫星的天线PCV文件。

### pos1-posopt2 = off，on（Rec PCV）

设置是否对接收机天线相位中心进行改正。如果设置为`on`，则需要在`Options`->`Files`下给出接收器天线PCV文件的路径，并在`Options`->`Positions`选项页面分别说明基站和流动站的接收器天线类型。IGS提供的天线文件中只包含测量型天线的，所以如果说你使用天线在这个文件列表里的话直接就可以使用。它（指的是否进行天线相位中心改正，译者注）主要影响z方向上的精度，所以如果对高程比较关注的话，则对天线相位中心进行改正就变得比较重要了。如果两个接收机的天线相同则可以置为`off`，因为它（通过差分的方式）会被消除。

### pos1-posopt5 = off，on（RAIM FDE）

如果有卫星的残差超过了阈值，则排除该卫星。这里只会排除那些误差比较大的卫星，但由于运算量较大，所以我通常会将这个选项禁用。

### POS1-exclsats =

如果你知道某颗卫星质量较差就把它放在这个列表里，那么它就不会参与到解算中来。除非在调试的时候的我怀疑某颗卫星有问题否则我很少会使用这个选项。

### pos1-navsys = 7,15，

因为更多的观测信息通常能带来更好的结果，所以（除常用的GPS系统外）通常我也会使用GLONASS和SBAS卫星。如果使用M8T较新的u-blox 3.0 固件，我会把Galileo系统也加入进来。

 

## SETTING2：

### pos2-armode =continuous, fix-and-hold

整周模糊度解算策略。`continuous`模式不利用固定解来调整相位偏差状态，所以它对假修复最不敏感。`fix-and-hold`则会利用固定解的反馈来帮助跟踪模糊度。我更喜欢使用`fix-and-hold`并将跟踪增益（pos2-varholdamb）调整得足够低，以最大限度地减少错误固定的可能性。如果`armode`未设置为`fix-and-hold`，则下面提到的和hold相关的任何选项都不再适用，包括pos2-gloarmode。

### pos2-varholdamb = 0.001,0.1（meter）

在`demo5`的代码中，可以使用此参数来调整`fix-and-hold`的跟踪增益。它实际上是方差而不是增益，因此设置较大的值时会产生较低的增益。0.001是默认值，任何超过100的效果都很小。该值用作在保持期间产生的伪测量的方差，其提供反馈以将卡尔曼滤波器中的偏置状态朝向整数值驱动。我发现从0.1到1.0的值提供足够的增益来辅助跟踪，同时在大多数情况下仍然避免跟踪错误修复。

### pos2-gloarmode = on，fix-and-hold，autocal

GLONASS整周模糊度解算模式。如果你有多个相同的接收机，那么通常你可以将该值设置为“on”，对于你来说这是一个比较好的选择，这样就可以允许在开始固定模糊度的时候用上glonass的卫星。如果你手里的接收机不一样或者你正在使用两个u-blox M8N接收机，这时候如果你想在模糊度解算中用上GLONASS卫星，这时候需要将此参数设置为“fix-and-hold”，从而消除信道间偏差。 。在这种情况下，刚开始模糊度固定的时候不会用到glonass卫星，等到（用其他系统的卫星）达到第一次固定，这时候可以对信道间偏差进行校正，待完成这一步后就可以用glonass的卫星进行模糊度固定。这里还有一个“autocal”选项，但我从来没有能够在2.4.3代码中顺利跑成功过。我把这个功能加到了demo5的代码中，这样就可以预先调整信道偏差，方差和校准增益。对于特定的接收机对，这个偏差值已知，所以可以在这里直接设置偏差值，同时可以将增益设置得非常低。这会破坏该功能的自动校准方面，但确实提供了一种机制来指定RTKLIB中缺少的偏差。当使用“autocal”时，GLONASS卫星将用于初始采集。“自主”功能还可用于使用迭代方法确定具有零或短基线的通道间偏差。这会破坏该功能的自动校准方面，但确实提供了一种机制来指定RTKLIB中缺少的偏差。当使用“autocal”时，GLONASS卫星将用于初始采集。“自主”功能还可用于使用迭代方法确定具有零或短基线的通道间偏差。这会破坏该功能的自动校准方面，但确实提供了一种机制来指定RTKLIB中缺少的偏差。当使用“autocal”时，GLONASS卫星将用于初始采集。“自主”功能还可用于使用迭代方法确定具有零或短基线的通道间偏差。

### POS2-gainholdamb = 0.01

在demo5代码中，可以使用此参数调整GLONASS卫星的通道间偏置校准的增益。 

### pos2-arthres = 3

这是用于确定模糊度解析解决方案是否有足够的信心来声明修复的阈值。它是第二最佳解决方案的残差平方与最佳解决方案的比率。我通常总是将其保留为默认值3.0并调整所有其他参数以解决此问题。虽然较大的AR比率表示比低AR比率更高的置信度，但两者之间没有固定的关系。卡尔曼滤波器状态中的误差越大，对于给定的AR比率，该解决方案的置信度将越低。通常，卡尔曼滤波器中的误差在第一次收敛时将是最大的，因此这是最有可能获得错误修复的时间。减少pos2-arthers1可以帮助避免这种情况。  

### pos2-arfilter = on

将此设置为开启将使从周期滑动中恢复的新坐姿或坐姿有资格。如果sat在第一次添加时显着降低AR比率，则其用于模糊度解析的使用将被延迟。启用此选项应允许您减少“arlockcnt”，它具有类似的用途，但具有盲目延迟计数。

###  pos2-arthres1 = 0.004-0.10

整数模糊度分辨率被延迟，直到位置状态的方差达到该阈值。它旨在避免在卡尔曼滤波器中的偏置状态有时间收敛之前的错误修复。如果已将eratio1设置为大于100的值或使用单个星座解决方案，则将此值设置为相对较低的值尤为重要。如果您看到AR比率为零延伸到您的解决方案中太远，您可能需要增加此值，因为这意味着已禁用歧义解决方案，因为尚未满足阈值。我发现0.004到0.10通常对我来说效果很好但是如果你的测量质量较低，你可能需要增加它以避免在发生多次循环滑动后过度延迟第一次修复或丢失修复。

### POS2-arthres2

相对于每个频率槽的GLONASS硬件偏差（米）。此参数仅在pos2-gloarmode设置为“autocal”时使用，用于指定两个不同接收器制造商之间的通道间偏置。要查找常见接收器类型的适当值，以及如何使用此参数进行迭代搜索以查找未指定的接收器类型的值，请参阅此文章。此参数已在RTKLIB 2.4.3中定义但未使用

### pos2-arthres3 = 1e-9,1e-7

GLONASS硬件偏置状态的初始方差。仅当pos2-gloarmode设置为“autocal”时才使用此参数。较小的值将给予pos2-arthres2中指定的初始值更多的权重。当pos2-arthres2被设置为已知偏差时我使用1e-9，而对于迭代搜索则使用1e-7。此参数已在RTKLIB 2.4.3中定义但未使用

### pos2-arthres4 = 0.00001,0.001

用于GLONASS硬件偏置状态的卡尔曼滤波器处理噪声。较小的值将给予pos2-arthres2中指定的初始值更多的权重。当pos2-arthres2设置为已知偏差时我使用0.00001，而迭代搜索设置0.001。此参数已在RTKLIB 2.4.3中定义但未使用

### pos2-arlockcnt = 0,5  

在将其用于整数模糊度解析之前延迟新的sat或sat从循环滑移中恢复的样本数。避免AR比率的腐败，包括尚未有时间收敛的sat。与“arfilter”结合使用。请注意，单位是样本，而不是时间单位，因此如果更改流动站测量采样率，则必须对其进行调整。对于u-blox接收器，我通常将其设置为零，这些接收器非常擅长标记可疑观察但是将其设置为至少为其他接收器的五个。如果不使用demo5 RTKLIB代码，请将其设置得更高，因为不支持“arfilter”功能。

### pos2-minfixsats = 4

获得修复所需的最小sats数。用于避免来自极少数卫星的错误修复，特别是在频繁循环滑动期间。

### pos2-minholdsats = 5

保持整数模糊结果所需的最小sats数。用于避免极少数卫星的误保持，特别是在频繁循环滑动期间。

### pos2-mindropsats = 10

每个时期允许将单个卫星排除在模糊度解决之外所需的最小sats数。在每个时代，排除不同的卫星。如果排除卫星导致AR比率的显着改善，那么该卫星将从用于AR的卫星列表中移除。

### pos2-rcvstds =on，off

启用此功能会导致原始伪距和相位测量观测值的测量方差根据接收器报告的测量值的标准偏差进行调整。目前仅u-blox接收器支持此功能。方差调整是基于stats-errphaseel参数对卫星高程进行的调整的补充。关闭后，我通常会得到更好的结果。

### pos2-arelmask = 15

功能上与默认值零没有区别，因为小于“elmask”的高程不会用于模糊度解析，但我改变它以避免混淆。

### pos2-arminfix = 20-100（5-20 *采样率）

保持模糊度所需的连续修复样本数。增加这可能是减少错误保持的最有效方法，但也会增加第一次保持的时间和重新获取保持的时间。随着模糊度跟踪增益减小（即随着pos2-varholdamb增加），观察数量增加，可以减少arminfix。请注意，如果流动站测量采样率发生变化，也应调整此值。

### pos2 elm mask = 15

功能上与默认值零没有区别，因为小于“elmask”的高程不会用于保持模糊度解析结果，但我改变它以避免混淆。

### pos2-aroutcnt = 100（20 *采样率）

连续丢失样本的数量，这些样本将导致重置歧义。同样，如果流动站测量采样率发生变化，则需要调整该值。

### pos2-maxage = 100

流动站测量和基准测量之间的最大延迟（差分年龄），以秒为单位。这通常是因为行为不当的无线电链路缺少测量结果。我已经从默认值中增加了它，因为我发现即使这个值相当大，我也经常会得到好的结果，假设在第一次修复并保持之后发生了丢失。

### pos2-rejionno = 1000

如果测量的预拟合残差大于此值（以米为单位），则拒绝测量。我发现RTKLIB不能很好地处理异常值测量，因此我将其设置得足够大以有效地禁用它。对于通常不擅长标记异常值的非ublox接收器，我有时必须将其设置回默认值30或甚至更低以尝试处理异常值，但这是一种权衡，因为它可能会导致其他问题特别是卡尔曼滤波器的初始收敛。

 

## OUTPUT：

### out-solformat = enu，llh，xyz

我通常对流动站和基地之间的相对距离感兴趣，因此将其设置为“enu”。如果您对绝对位置感兴趣，请将其设置为“llh”，但请确保在“ant2”设置中设置确切的基本位置。如果需要精确的z轴测量，请注意此设置。如果流动站处于恒定高度，只有llh格式会给你一个恒定的z高度。“Enu”和“xyz”是笛卡尔坐标，因此z轴遵循平面，而不是地球的曲率。如果基站位于离流动站更远的位置，这会导致特别大的误差，因为曲率将随着距离而增加。

###  out-outhead = on

解决方案没有功能差异，只需向结果文件输出更多信息。

### out-outopt = on

解决方案没有功能差异，只需向结果文件输出更多信息。

### out-outstat =residual

解决方案没有功能差异，只是将残差输出到文件。残差对于调试解决方案的问题非常有用，只要残差文件与解决方案文件位于同一文件夹中，就可以使用RTKPLOT绘制残差。  

### stats-eratio1 = 300，stats-eratio2 = 300

伪距测量的标准偏差与载波相位测量的比率。我发现较大的值对于低成本接收器更有效，但是默认值100通常对于更昂贵的接收器更好，因为它们具有较少的噪声伪距测量。较大的值往往会导致卡尔曼滤波器更快收敛并导致更快的首次修复，但也会增加错误修复的可能性。如果你增加这个值，你应该将pos2-arthres1设置得足够低，以防止在卡尔曼滤波器有时间收敛之前找到修复。我相信增加该值对于增加伪距平滑算法的时间常数具有类似的效果，因为它在保持低频分量的同时滤除伪距测量中的更多较高频率。

###  stats-prnaccelh = 3.0

如果启用了接收器动态，则使用此值设置水平分量中流动站接收器加速度的标准偏差。该值应包括所有频率的加速度，而不仅仅是低频率。它应该表征流浪者天线的任何运动，而不仅仅是整个流动站的运动，因此它可能比你想象的要大。它将包括来自振动的加速度，道路颠簸等，以及整个漫游车的更明显的刚体加速度。可以通过运行此值设置为较大值的解决方案来估计，然后使用RTKPLOT检查解决方案文件中的加速值

### stats-prnaccelv = 1.0

关于水平加速度的评论更多地适用于垂直加速度分量，因为在许多应用中，有意加速都将在水平分量中。最好从实际GPS测量数据中获得该值，而不是刚体流动站的预期值。最好高估这些值而不是低估它们。

### ant2-postype = rinexhead，llh，single

这是基站天线的位置。如果您只对基座和流动站之间的相对距离感兴趣，则此值不需要特别准确。对于后处理，我通常使用RINEX文件头中的近似基站位置。如果您想在解决方案中保持绝对位置，那么基站位置必须更准确，因为任何错误都会增加您的流动站位置错误。如果我想要绝对位置，我首先处理附近参考站的基站数据以获得确切位置，然后使用“llh”或“xyz”选项指定该位置。对于实时处理，我使用“单个”选项，该选项使用来自数据的单个解决方案来粗略估计基站位置。

### ant2-maxaveep = 1

如果“postype”设置为“single”，则指定平均以确定基站位置的样本数。我将其设置为1以防止卡尔曼滤波器开始收敛后基站位置发生变化，因为这似乎导致首次修复很长时间。在大多数情况下进行后处理时，基站位置将来自RINEX文件头，因此您不会使用此设置。但是，如果您正在使用RTCM文件，甚至可能需要进行后期处理。

 

## MISC：

### misc-timeinterp = off，on

插值基站观测值。如果基站观测采样时间大于5秒，我通常将其设置为“on”。

如果您已成功调整其他选项或使用这些选项的不同设置，或者您不同意我的任何建议，请帮助我更新此列表。我将此视为一份工作文件，并在我了解更多信息后继续更新。