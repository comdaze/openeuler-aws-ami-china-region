# openEuler AWS

使用 `euler-packer` 脚本为 [AWS](https://aws.amazon.com/) 构建 openEuler [AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html) 镜像。

默认会在 AWS `cn-northwest-1` (宁夏) 和 `cn-north-1` (北京) 区域生成 AMI 镜像，可通过修改 [openeuler/aws/](/openeuler/aws/) 目录下的 Packer 配置文件指定 AMI 的生成区域。

![](./docs/images/openeuler/generated-ami.png)

## 构建流程

`euler-packer` 脚本构建 openEuler AWS AMI 镜像的流程如下：

1. 构建基础镜像 (base-image)

    1. 下载 openEuler qcow2 格式的虚拟机镜像至本地。
    1. 使用 `qemu-nbd` 将 qcow2 格式的虚拟机镜像分区加载至系统，将总大小为 40G 的磁盘分区调整为 8G。

1. 构建基础 AMI 镜像 (base-ami)

    1. 将调整过分区大小的 qcow2 镜像转换为 RAW 格式，上传至 AWS S3 存储桶。
    1. 将存储桶中的 RAW 镜像创建 Snapshot，之后使用此 Snapshot 创建基础 AMI 镜像 (`DEV-*-BASE`)。

    > 基础 AMI 镜像不包含 `cloud-init` 机制，且未禁用 root 密码登录，基础 AMI 镜像仅用来构建最终的 AMI 镜像或调试使用。<br>
    > 在完成镜像构建后，可删除 DEV 基础 AMI 镜像和 Snapshot，减少不必要的费用开销。

1. 使用 Packer 构建 AMI 镜像

    1. 使用 Packer 启动 “基础 AMI 镜像” EC2 虚拟机。
    1. 在虚拟机中安装 `cloud-init` 等基础软件包，调整内核参数，删除 root 密码等。
    1. 最终将此 EC2 虚拟机的磁盘制作 Snapshot，并制作最终可供使用的 AMI 镜像 (`openEuler-<VERSION>-<ARCH>-hvm-<DATETIME>`)。

## 准备工作

1. Linux 系统

    在构建镜像的过程中，需要使用 `qemu-nbd` 将 qcow2 格式的镜像的分区表加载至系统中，之后对根分区进行缩容和分区表调整，因此 `euler-packer` 的脚本仅支持在 Linux 系统上运行。

    > 本仓库脚本使用系统 ubuntu 24.04

1. 安装依赖

    安装运行脚本所需的依赖：`git`,`docker`, `awscli`, `jq`, `qemu-utils`, `partprobe` (`parted`), `packer`, `fdisk`

    ```sh
    sudo apt update && sudo apt upgrade -y
    sudo apt install curl unzip -y
    sudo apt install git jq qemu-utils parted fdisk util-linux -y
    #安装awscli
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install
    ```

    本仓库的脚本所使用的 Packer 版本需要大于等于 1.7，请按照 [官方下载](https://developer.hashicorp.com/packer/install) 可以采用Binary download 进行安装 Packer。
    ```sh
    wget https://releases.hashicorp.com/packer/1.11.2/packer_1.11.2_linux_amd64.zip
    unzip packer_1.11.2_linux_amd64.zip 
    sudo mv packer /usr/bin
    ```
    手动安装 amazon ebs packer 依赖插件，可以在[packer-plugin-amazon链接](https://github.com/hashicorp/packer-plugin-amazon/releases)下载

    ```sh
    mkdir amazon && cd amazon
    wget https://github.com/hashicorp/packer-plugin-amazon/releases/download/v1.3.3/packer-plugin-amazon_v1.3.3_x5.0_linux_amd64.zip
    unzip packer-plugin-amazon_v1.3.3_x5.0_linux_amd64.zip
    packer plugins install --path ./packer-plugin-amazon_v1.3.3_x5.0_linux_amd64 github.com/hashicorp/amazon
    ```

1. 初始化 `awscli` 并配置环境变量

    - 执行 `aws configure`，填写 Access Key ID，Secret Key 并将区域设定为 `ap-northeast-1`。
    - 设定环境变量 `AWS_ACCESS_KEY_ID`，`AWS_SECRET_ACCESS_KEY`。

    ```sh
    # Generate ~/.aws/credential
    aws configure
    # Set environment variables
    export AWS_ACCESS_KEY_ID=<access_key_id>
    export AWS_SECRET_ACCESS_KEY=<secret>
    ```

1. 建立 [AWS S3 存储桶](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)

    在构建镜像的过程中会将 RAW 格式的 openEuler 镜像上传至此存储桶中，之后为存储桶中的 RAW 镜像创建 Snapshot。

1. 克隆此仓库代码

    ```sh
    git clone https://github.com/comdaze/openeuler-aws-ami-china-region.git && cd openeuler-aws-ami-china-region
    ```

1. 其他

    构建镜像时，脚本会使用 `date +"%Y%m%d"` 获取时间为 AMI 镜像命名，因此请确保运行此脚本的系统时间和时区设置正确。

## 构建 AMI 镜像

```bash
./openeuler.sh \
    --aws \
    --aws-bucket <BUCKET_NAME>
    --aws-owner-id <OWNER_ID> \
    --version "24.03-LTS" \
    --arch "x86_64" \
```

执行脚本的参数：

- `--aws-bucket`: AWS S3 存储桶名称（**必须**）
- `--aws-owner-id`: AWS 帐号的 Owner ID，当前帐号的 Owner ID 可从 AWS 控制台获取（**必须**）
- `--version`: openEuler 版本号（**必须**）
- `--arch`: 系统架构，默认为 `x86_64`，可设定为 `x86_64` 或 `aarch64`
- `--mirror`: 下载 openEuler qcow2 镜像的镜像源链接，默认为 `https://repo.openeuler.org`

----

最终构建的 AMI 镜像的命名格式为 `openEuler-<VERSION>-<ARCH>-hvm-<DATETIME>`。

## 其他

1. `24.03-LTS` 版本 `aarch64` 架构的 Linux 内核未包含 [ENA](https://github.com/amzn/amzn-drivers/tree/master/kernel/linux/ena) 网卡驱动，在构建此版本 `aarch64` 架构的 openEuler 镜像时会下载已编译好的 ENA 网卡驱动至 `/opt/ena-driver/ena.ko`，并配置 `modprobe` 在开机时自动加载此网卡驱动。

    在使用 AMI 镜像时请勿删除此文件，否则将无法连接至网络。

1. openEuler 一键安装高版本 Docker 脚本 [scripts/others/install-docker.sh](/scripts/others/install-docker.sh)。


## Docker Installation

Script to install Docker for openEuler: [scripts/others/install-docker.sh](/scripts/others/install-docker.sh)

## License

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

[http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0)

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
