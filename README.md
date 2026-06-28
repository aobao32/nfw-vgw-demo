# 使用 AWS Network Firewall 服务审查 IDC 和云上 VPC 间的流量 - VGW 架构的设计

本实验搭建了一个云上 VPC 和 模拟IDC 环境的网络环境，通过开启 BGP 路由传播并手工配置高优先级路由条目，验证 IDC 和云之间的网络流量经 NFW 审查的场景。

全文大量使用的技术术语缩写/简称如下：

| 全称 | 简称 |
| --- | --- |
| Network Firewall | NFW |
| Internet Gateway | IGW |
| NAT Gateway | NAT |
| Transit Gateway | TGW |
| Direct Connect | DX |
| CloudFormation | CFN |
| Virtual Private Gateway | VGW |
| Border Gateway Protocol | BGP |

在云上创建好的资源名称使用如下简写：

| 全称 | 简称 |
| --- | --- |
| Availability Zone | az |
| Route Table | rt |
| Private | pvr |
| Network Firewall | fw |

## 一、验证 NFW 流量审查部署环境的挑战

### 1、DX 专线使用 VGW 和 TGW 的场景选择

IDC 和 云的连接通常是 DX 专线连接，在专线与 VPC 互联时候，有两种不同的技术组件连接，分别是单个 VPC 时候的 VGW 和多个 VPC 时候的 TGW 两种配置方式。由于 TGW 有 attachment 小时费和数据处理费，而 VGW 自身不产生附加流量处理费，因此很多用户会优先使用 VGW 连接到 VPC 以降低成本。使用 VGW 虽然没有互联流量处理成本，但是其路由表管理复杂度更高，因为 VPC 之间不能进行转发，所以需要多条点对点的互联。本文的场景假设云端只有一个业务 VPC 到 IDC 之间的流量需要审查，因此 VGW 方案完全足够。

使用 VGW 互联时候，通常会开启 BGP 的路由传播功能，让云上 VPC 可以自动学习到 IDC 路由器通告过来的子网路由，由此无须手工填写 IDC 网段的路由条目，即可实现云上和云下互通。

### 2、云和 IDC 之间的流量审查实现原理

在云和 IDC 连通后，BGP 宣告的路由是最短路径，从云上下一跳去往 VGW，即可到达 IDC。而启用 NFW 检测后，下一跳需要修改为 NFW Endpoint。此时，流量走向和 IDC 的 BGP 宣告的路由条目会产生差别。为解决路由表的差异，有两种办法。

第一个办法是关闭云上 VPC 的 BGP 路由传播学习功能，不让云上 VPC 自动学习到前往 IDC 的路由。然后手工填写去往 IDC 的所有路由条目。这样做的主要挑战是在 IDC 有许多子网的情况下，逐一手工填写路由表会非常麻烦，容易错配或者漏配，多个子网路由表还需要多次重复配置，效率低下。因此关闭云上 VPC 的 BGP 路由传播学习功能不是最佳选择。

第二个办法，是保持云上 VPC 的 BGP 路由传播学习功能，云上 VPC 始终接收到模拟 IDC 网络环境传播过来的路由条目。然后手工填写优先级更高的路由条目，将流量送往 NFW 进行检查。当 VPC 路由表有多个条目的 CIDR 有交集的时候，将按照如下原则匹配优先级：

- 更长的子网掩码（更精确的匹配）优先选择路由
- 子网掩码长度相同时，手写静态路由优先级高于自动传播学习过来的路由条目

以上两个原则确保了手工输入的路由表优先级更高，因此使得流量将优先经过 NFW 检测，从而实现无须关闭BGP传播就能控制将特定网段的流量发给 NFW 检测的效果。

### 3、使用 Site-to-site VPN + VGW 模拟 DX 专线 + VGW 的架构

由于 DX 专线服务的配置要求客户路由器接入 DX Location 形成物理 Connection，并配置 VIF，由此导致做实验和技术验证的困难很高，需要真实环境才能模拟。本文使用 Site-to-site VPN + VGW 作为替代方案来模拟 DX 专线环境。Site-to-Site VPN 既可挂在 VGW 上，也可挂在 TGW 上，本方案选择 VPN 挂在 VGW 上，与真实 DX + VGW 场景的 BGP 路由传播行为一致。由此，可将其作为本方案在路由表行为层面的模拟环境。

本文通过一个 CloudFormation 模板快速拉起这个环境，拉起后首先验证 BGP 网络宣告成功以及路由通畅，然后手工切换路由表将流量发送到 NFW，验证 NFW 的规则工作正常。

## 二、设计并启动整个模拟实验环境

### 1、网络架构设计

架构图如下：

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-vgw-dx-architecture.png)

在云上用一个独立的 VPC 模拟 IDC ，使用 site-to-site VPN 和 BGP 协议进行互联，并开启了路由传播。当 CFN 启动成功后，路由是自动传播的，流量默认走 VGW ，不经过 NFW。当配置 NFW 的路由后，流量将通过 NFW 子网，扫描完成后与 IDC 形成互通。

关于互联网出口的补充：IDC 网络和云端 VPC 网络都使用模拟 IDC 网络的 Router 作为唯一的互联网出口，这个 Router 既承担了去互联网流量的 SNAT，又承担了 Site-to-site VPN 的网关。当然，去往 Site-to-site VPN 的流量不做 SNAT，网关只负责建立连接。

### 2、使用 CFN 模板启动实验环境

以上网络架构使用 CFN 模板实现，这里选择 CFN 而不是 AWS CDK，因为 CFN 更加简单，用户无需安装 AWS CDK 等开发包，这对进行网络测试而并非拥有完整 AWS 开发工具的网络用户更加友好。

本项目 CloudFormation 在 Github 地址：[https://github.com/aobao32/nfw-vgw-demo/tree/main/cloudformation](https://github.com/aobao32/nfw-vgw-demo/tree/main/cloudformation)

首先进入 CloudFormation 服务界面，点击创建 Stack，选择上传模板。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-01.png)

输入模板名称后，这里要选择的是在本区域的 EC2 密钥，如果在本区域还没有 EC2 登录密钥，需要事先创建好。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-02.png)

选择允许创建 IAM 权限。点击下一步继续。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-03.png)

Review 所有选项，点击下一步完成。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-04.png)

等待大约15-20分钟，创建完成。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-05.png)

注意：即便模板状态显示为绿色，也还需要再等待5-10分钟，等待所有网络协议启动成功。这是因为 CFN 不会等待 VPN 隧道 UP 和 BGP 会话建立后才把 Stack 标记为完成。

### 3、验证网络联通

CFN模板创建发起到完成，中间启动多种服务，大约需要15-20分钟。检查服务完全启动成功的标准是：

- 1) AWS托管服务 Site-to-Site VPN 里边，两条 VPN 隧道状态都显示 UP，BGP 会话建立成功
- 2) 云上VPC的 `prv-rt-az1` 和 `prv-rt-az2` 两个路由表，里边学习到了 BGP 传播过来的路由
- 3) 云上 VPC 的 EC2 和模拟 IDC 的 EC2 互 ping 成功

由此分别验证以上步骤。

首先进入 VPC 服务，查看 Site-to-site VPN 服务，点击 VPN ID。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-06.png)

查看 VPN 隧道，确认里边显示的状态是 UP 。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-07.png)

现在切换到 VPC 服务，进入路由表界面，找到标签名为 `prv-rt-az1` 路由表，点击路由条目标签页，查看其中的路由传播 `Propagated` 字段，确认其中显示为 `Yes`。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-08.png)

现在进入 EC2 服务，找到 VPC 云上 Workload 的 EC2 虚拟机，选中后，点击右上角 `Connect` 连接按钮。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-09.png)

在弹出窗口中，选择第二个标签页 `SSM Session Manager` 标签页，然后点击右下角连接按钮。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-10.png)

由此即可登录到 EC2 。

登录 EC2 成功后，从云上 VPC 的 EC2 上，使用 ping命令/curl命令 测试连接到 模拟 IDC 的 EC2，测试连通成功。测试通过后，进入下一步 NFW 路由表的配置。

## 三、手工配置路由将流量送给 NFW

### 1、手工配置路由表

手工切换流量，让流量经过 NFW：

- 1) 从云上到 IDC方向：修改 `prv-rt-az1` 和 `prv-rt-az2`，显式增加路由条目，将去往 IDC 的流量发送到 NFW Endpoint
- 2) 从IDC到云上方向：修改 `vgw-ingress-rt` 路由表，显式增加路由条目，将去往 云上VPC 的流量发送到 NFW Endpoint

现在开始配置。首先找到 `prv-rt-az1` 路由表，添加去往 IDC 的路由，下一跳选择 AZ1 的 NFW Endpoint。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-11.png)

对另一个 AZ 的路由表 `prv-rt-az2` 也做对应配置，下一跳选择 AZ2 的 NFW Endpoint。

最后编辑从 IDC 到 云上的路由表，即 `vgw-ingress-rt` 这张表，在里边分别填写 az1 和 az2 两个子网目标地址，下一跳分别选择两个 AZ 的 NFW Endpoint。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-12.png)

完成以上配置后，流量会经过 NFW 检测。

### 2、NFW 规则设置

NFW 的设置可以在左侧的 NFW 菜单下，Policy 策略菜单中看到的 Stateful Policy 有状态组的规则。本方案的 Stateful 规则组采用 `STRICT_ORDER` 严格顺序模式，其中序号更小的是优先执行，序号更大的是最后执行，因此组合起来就是拦截HTTP请求、除此外放行所有规则。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-16.png)

### 3、验证网络连通性和 NFW 流量检查

首先测试云到 IDC方向。在 EC2 界面，通过 SSM Session Manager 登录到 EC2，从云上 EC2 访问代表 IDC 的 EC2 内网 IP，ping和curl访问都正常。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-13.png)

接下来测试 IDC 到云方向。从代表 IDC 的 EC2，对云上 VPC 的 EC2 内网 IP 地址，发起 ping 命令可连通，但是 curl 访问会被拦截。这是 NFW 的预期行为，通过 CFN 模板启动的环境，NFW 预设规则会拦截从 IDC 到 云上VPC 方向 curl 访问。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-14.png)

### 4、通过 CloudWatch 日志查看 NFW 对流量的检测和拦截

本 CFN 创建的 NFW，开启了日志功能，打开了 Flow 流量日志和 Alert 告警/匹配日志。进入 CloudWatch 的 LogGroup日志组，查看名为：

- nfw-log-flow：NFW 流量日志
- nfw-log-alert：NFW 触发 alert 或 drop 动作的规则匹配事件日志

即可确认流量被正常扫描。

上一章节我们测试了从 云上 VPC 方向到 IDC 方向，以及 IDC 方向到云上 VPC 方向，现在查看日志验证 NFW 行为符合预期。如下截图。

![](https://blogimg.bitipcman.com/workshop/nfw/nfw-15.png)

至此全部测试结束。

## 四、小结

通过以上实验，可以获得两个结论：

- 1、针对 VGW 和 BGP 传播的测试，可以使用 AWS 托管的 Site-to-site VPN 建立 BGP 连接，由此可以开启 BGP 传播，并测试路由表配置
- 2、VPC 路由表中，在子网掩码长度相同的情况下，手工输入的静态路由优先级比自动传播的优先级更高

## 五、参考官方文档

- 带有Internet Gateway和NAT Gateway的架构：https://docs.aws.amazon.com/network-firewall/latest/developerguide/arch-igw-ngw.html