---
title: "离线安装"
weight: 60
---
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

你可以使用两种不同的方法在离线环境中安装 K3s。离线环境指的是不直接连接到 Internet 的环境。你可以部署私有镜像仓库和 mirror docker.io，也可以手动部署镜像，例如用于小型集群。

## 私有镜像仓库

本文档假设你已经在离线环境中创建了节点，并且在堡垒主机上拥有 Docker 私有镜像仓库。

如果你尚未设置私有 Docker 镜像仓库，请参阅[官方文档](https://docs.docker.com/registry/deploying/#run-an-externally-accessible-registry)。

### 创建镜像仓库 YAML

按照[私有镜像仓库配置](private-registry.md)指南创建和配置 registry.yaml 文件。

完成此操作后，你现在可以转到下面的[安装 K3s](#安装-k3s) 部分。


## 手动部署镜像

我们假设你已经在离线环境中创建了节点。
此方法需要你手动将必要的镜像部署到每个节点，适用于无法运行私有镜像仓库的边缘部署。

### 准备镜像目录和 K3s 二进制文件
从 [Releases](https://github.com/rancher/k3s/releases) 页面获取要运行的 K3s 版本的镜像 tar 文件。

将 tar 文件放在 `images` 目录下，例如：

```bash
sudo mkdir -p /var/lib/rancher/k3s/agent/images/
sudo cp ./k3s-airgap-images-$ARCH.tar /var/lib/rancher/k3s/agent/images/
```

将 K3s 二进制文件放在 `/usr/local/bin/k3s` 中，并确保文件是可执行的。

按照下一节中的步骤安装 K3s。

## 安装 K3s

### 先决条件

- 在安装 K3s 之前，完成上面的[私有镜像仓库](#私有镜像仓库)或[手动部署镜像](#手动部署镜像)操作，预填充 K3s 需要安装的镜像。
- 从 [Releases](https://github.com/rancher/k3s/releases) 页面下载 K3s 二进制文件，该文件需要匹配用于获取离线镜像的版本。将二进制文件放在每个离线节点上的 `/usr/local/bin` 中，并确保文件是可执行的。
- 在 https://get.k3s.io 下载 K3s 安装脚本。将安装脚本放在每个离线节点上的任何位置，并将其命名为 `install.sh`。

使用 `INSTALL_K3S_SKIP_DOWNLOAD` 环境变量运行 K3s 脚本时，K3s 将使用脚本的本地版本和二进制文件。


### 在离线环境中安装 K3s

你可以在一台或多台服务器上安装 K3s，如下所述。

<Tabs>
<TabItem value="单节点配置" default>

要在单个服务器上安装 K3s，只需在 Server 节点上执行以下操作：

```bash
INSTALL_K3S_SKIP_DOWNLOAD=true ./install.sh
```

然后，如果要添加其他 Agent，请在每个 Agent 节点上执行以下操作。请确保将 `myserver` 替换为服务器的 IP 或有效 DNS，并将 `mynodetoken` 替换为服务器的节点令牌（通常位于 `/var/lib/ rancher/k3s/server/node-token`）。

```bash
INSTALL_K3S_SKIP_DOWNLOAD=true K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken ./install.sh
```

</TabItem>
<TabItem value="高可用配置" default>

参考[具有外部数据库的高可用](ha.md)或[具有嵌入式数据库的高可用](ha-embedded.md)指南。你需要调整安装命令来指定 `INSTALL_K3S_SKIP_DOWNLOAD=true`，并在本地运行安装脚本，而不是使用 curl。你还将使用 `INSTALL_K3S_EXEC='args'` 为 K3s 提供参数。

例如，具有外部数据库的高可用指南的第二步提到了以下内容：

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint='mysql://username:password@tcp(hostname:3306)/database-name'
```

你需要修改此类示例，如下所示：

```bash
INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_EXEC='server' K3S_DATASTORE_ENDPOINT='mysql://username:password@tcp(hostname:3306)/database-name' ./install.sh
```

</TabItem>
</Tabs>

> **注意**：K3s 还为 kubelet 提供了一个 `--resolv-conf` 标志，这可能有助于在离线网络中配置 DNS。

## 升级

### 安装脚本方法

你可以通过以下方式完成离线环境的升级：

1. 从 [Releases](https://github.com/rancher/k3s/releases) 页面下载要升级的 K3s 版本的新离线镜像 tar 包。将 tar 文件放在每个节点上的 `/var/lib/rancher/k3s/agent/images/` 目录中。删除旧的 tar 文件。
2. 复制并替换每个节点上 `/usr/local/bin` 中的旧 K3s 二进制文件。复制 [K3s 安装脚本](https://get.k3s.io)（因为脚本可能自上次版本发布以来已更改）。使用相同的环境变量再次运行脚本。
3. 重启 K3s 服务（如果安装程序没有自动重启 K3s 的话）。


### 自动升级

从 v1.17.4+k3s1 开始，K3s 支持[自动升级](../upgrades/automated.md)。要在离线环境中启用此功能，你必须确保所需的镜像在你的私有镜像仓库中可用。

你将需要与你打算升级到的 K3s 版本相对应的 rancher/k3s-upgrade 版本。注意，镜像标签将 K3s 版本中的 `+` 替换为 `-`，因为 Docker 镜像不支持 `+`。

你还需要在你要部署的 `system-upgrade-controller` 清单 YAML 中指定的 `system-upgrade-controller` 和 `kubectl` 版本。在[这里](https://github.com/rancher/system-upgrade-controller/releases/latest)检查 `system-upgrad-controller` 的最新版本，并下载 `system-upgrade-controller.yaml` 来确定你需要推送到私有镜像仓库的版本。例如，在 `system-upgrade-controller` 的 v0.4.0 版本中，清单 YAML 中指定了这些镜像：

```
rancher/system-upgrade-controller:v0.4.0
rancher/kubectl:v0.17.0
```

将必要的 rancher/k3s-upgrade、rancher/system-upgrade-controller 和 rancher/kubectl 镜像添加到私有镜像仓库后，请按照[自动升级](../upgrades/automated.md)指南进行操作。
