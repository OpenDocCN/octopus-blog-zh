# 八大集装箱登记处-部署八达通

> 原文：<https://octopus.com/blog/top-8-container-registries>

容器注册中心经常与它们的存储库副本混淆，尽管它们服务于不同的目的。

容器存储库是容器化应用程序映像的存储。如今，大多数图像存储库都专注于“OCI”格式，基于 Docker 普及并向所有人开放的容器格式。事实上，“OCI 形象”和“码头形象”在注册服务商的营销中经常互换使用。

OCI 主张开放集装箱倡议。该计划是一个容器结构，旨在充当行业标准格式。技术、开发和云服务领域的大多数主要参与者都支持这一倡议，并支持 OCI 格式。在[开放容器倡议网站](https://opencontainers.org/)上了解更多信息。

然后，容器*注册中心*既作为容器存储库的集合，又作为管理和部署映像的可搜索目录。

市场上有许多容器注册选项，为不同类型、大小和需求的客户提供服务。让我们看看我们的 8 强，以及他们为什么会有吸引力。

## 坞站集线器

鉴于 Docker 发明了用于容器交付的标准 OCI 格式，并被所有主要操作系统采用，因此 [Docker Hub](https://hub.docker.com/) 也是图像管理的标准注册中心是有道理的。如果你正在开发，很可能你已经使用过 Docker Hub，尤其是如果你曾经关注过我们的技术指南或博客。

虽然这个列表中的所有注册表服务都可以帮助你管理 Docker 的格式，但 Docker Hub 作为一个注册表仍然值得在这里占有一席之地。不仅仅是因为它提供了一个巨大的公共注册表，你可以用它来部署来自大大小小的供应商的应用，也可以交付你自己的应用。

## 亚马逊 ECR

由于其集成和团队管理选项，如果您已经在使用亚马逊网络服务(AWS)来托管应用程序，亚马逊的弹性容器注册中心(ECR) 会很有用。

Amazon 的 ECR 融入了他们所有的容器托管服务，因此您可以轻松地管理和推送您的图像到:

*   弹性集装箱服务
*   弹性立方结构服务(EKS)
*   哦，舔舔

(当然，前提是你能理解所有这些相似的缩写。)

像 Docker Hub 一样，他们的公共注册中心和市场也是物有所值的，允许部署许多产品、自由软件和开源项目。您永远不会缺少可以构建的现有环境。

## 海港

Harbor 是一个开源注册表，你几乎可以安装在任何地方，但特别适合 Kubernetes。

在某些服务将它们的注册中心与它们自己的服务紧密结合的情况下，Harbor 的自由使它成为一个多用途的选择。它与大多数云服务以及持续集成和持续交付(CI/CD)平台兼容，也是一个很好的内部解决方案。

## Azure 容器注册表

在向 AWS 提供类似服务的同时，微软的[Azure Container Registry(ACR)](https://azure.microsoft.com/en-au/services/container-registry/)也支持 Docker 和 OCI 图像，以及 Helm charts。

然而，Azure 最大的卖点是它的注册地理复制。这确保了每个人都能以他们习惯的速度访问图像，无论他们身在何处。

## GitHub 容器注册表

鉴于 GitHub 的影响力，并且它已经对所有用户可用， [GitHub 的容器注册表](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)(称为 [GitHub 包](https://github.com/features/packages)的更大功能的一部分)是最容易接近的选项之一。

考虑将 GitHub 包用于容器管理的好处包括:

*   简化的用户管理-您的大多数用户已经拥有帐户
*   GitHub Actions 集成推送、发布和部署图像
*   相对于其他服务的成本

## 谷歌容器注册

当其他服务将他们的承诺集中在对潜在客户子集重要的某些功能上时，[谷歌云的容器注册(GCR)](https://cloud.google.com/container-registry/) 是一个可靠的多面手。它做了你想从容器注册表中得到的一切，而且做得非常好。

就像云服务的其他大牌一样，你可能会被锁定在 GCR，这取决于你使用的谷歌云产品。例如，Google Cloud Run 将只使用存储在 GCR 的图像，所以在选择注册服务时要记住这一点。

它不像 AWS 或 Azure 那样吹嘘自己的功能，但谷歌云的 GCR 是云提供商“三巨头”之一的一个有价值的产品。

## JFrog 容器注册表

基于另一个 JFrog 产品 Artifactory， [JFrog 容器注册表](https://jfrog.com/container-registry/)支持 Docker 图像*和*舵图。JFrog Container Registry 还提供了存储任何包类型的选项，这要归功于它的通用存储库。

JFrog 既有云和自托管两种选择(或者混合，如果你愿意的话)，并承诺很好的可伸缩性。

## 红帽码头

与其他选择不同，[红帽码头](https://www.redhat.com/en/technologies/cloud-computing/quay)只提供私人集装箱注册。这使得它特别适合企业级客户。

Quay 不受云提供商限制，可轻松连接到 DevOps 管道两端的系统。像 Azure 一样，Quay 也包括地理位置选项，有趣的是，它支持 BitTorrent 进行集装箱配送。

Red Hat 还在其 Kubernetes 平台 OpenShift 中包含了一个精简的注册表解决方案。然而，他们确实建议大型团队和组织使用 Quay。

## 下一步是什么

在我们最近的一些帖子中，我们更多地讨论了容器化和云流程编排，包括:

愉快的部署！