# 识别 AWS 影子 IT 资源- Octopus 部署

> 原文：<https://octopus.com/blog/managing-aws-shadow-it>

术语“影子 IT”通常指由开发运维团队成员部署的临时 IT 资源，用于解决紧急需求，但缺乏支持长期基础架构的持续维护和管理。

在 AWS 中，影子 IT 采用虚拟机、S3 存储桶、虚拟专用云以及任何其他类型的动态创建的资源的形式，通常通过 web 控制台实现。

在这篇文章中，我解释了一些与 shadow IT 相关的问题，以及如何识别 AWS 帐户中的非托管资源。

## 影子 IT 的问题

假设您是 DevOps 团队的一名新成员，负责解决一个名为“Web 服务器”的 AWS 虚拟机故障。谁创建了这个虚拟机？它托管哪些应用程序？备份位于何处？如何重新创建虚拟机？

这些信息都无法从 VM 资源本身获得，特别是当它失败到无法再登录时。由于无法获得这些信息，几乎不可能提供支持。

任何未能围绕创建新的云资源实现标准实践的成长中的团队，很可能会在几年后发现自己不知道任何给定资源做了什么。这带来了安全挑战，因为不清楚谁负责像操作系统补丁这样的任务，或者当最初应用许可的“允许所有”规则集时，什么网络规则是合适的。这也使得很难知道资源的当前状态是否正确，或者是否有人意外地进行了不期望的更改。

在 AWS 中，描述资源属性(如谁拥有它以及它的用途)的第一步是应用一组标准化的标签。下一步是确保所有资源都是用声明性模板创建的，如 CloudFormation，它允许在需要时重新创建资源，还允许检测不需要的手动更改，即漂移。

为了识别不兼容的资源，您将创建两个简单的 runbooks 来扫描您的 AWS 帐户中没有预期标记的资源。

## 识别未标记的资源

下面的 Bash 脚本扫描整个 AWS 帐户，查找缺少任何所需标记的资源。在这个例子中，标签`Team`、`Deployment Project`和`Environment`指的是拥有资源的团队、部署它的 CI 或 Octopus 项目，以及资源所属的环境(比如开发或生产):

```
REQUIREDTAGS=("Team"  "Deployment Project"  "Environment")
OUTPUT=$(aws resourcegroupstaggingapi get-resources --tags-per-page 100)

for ((i = 0; i < ${#REQUIREDTAGS[@]}; i++)); do
    COUNT=$(echo ${OUTPUT} | jq -r "[.ResourceTagMappingList[] | select(contains({Tags: [{Key: \"${REQUIREDTAGS[$i]}\"} ]}) | not)] | length")
    echo "==========================================================="
    echo "The following ${COUNT} resources lack the ${REQUIREDTAGS[$i]} tag."
    echo "==========================================================="
    echo ${OUTPUT} | jq -r ".ResourceTagMappingList[] | select(contains({Tags: [{Key: \"${REQUIREDTAGS[$i]}\"} ]}) | not) | .ResourceARN"
done 
```

## 识别非托管资源

查找不是由 CloudFormation 模板创建的资源(即非托管资源)与上面的脚本非常相似。

几乎所有的 AWS 资源都支持标签，当这些资源由 CloudFormation 模板创建时，会自动应用`aws:cloudformation:stack-id`标签。这意味着识别非托管资源就像找到任何缺少`aws:cloudformation:stack-id`标签的资源一样简单。

下面的 Bash 脚本扫描所有资源，除了缺少`aws:cloudformation:stack-id`标签的 CloudFormation 栈:

```
OUTPUT=$(aws resourcegroupstaggingapi get-resources --tags-per-page 100)
COUNT=$(echo $OUTPUT | jq -r '[.ResourceTagMappingList[] | select(contains({Tags: [{Key: "aws:cloudformation:stack-id"} ]}) | not) | select(.ResourceARN | test("arn:aws:cloudformation:[a-z]+-[a-z]+-[0-9]+:[0-9]+:stack/.*") | not)] | length')

echo "==========================================================="
echo "The following ${COUNT} resources were not created by CloudFormation"
echo "==========================================================="
echo $OUTPUT | jq -r '.ResourceTagMappingList[] | select(contains({Tags: [{Key: "aws:cloudformation:stack-id"} ]}) | not) | select(.ResourceARN | test("arn:aws:cloudformation:[a-z]+-[a-z]+-[0-9]+:[0-9]+:stack/.*") | not) | .ResourceARN' 
```

## 解决不符合的资源

在缺少标记的资源上定义标记通常是通过 web 控制台或 CLI 手动添加标记。下面的脚本显示了向资源批量添加公共标记的示例:

```
aws resourcegroupstaggingapi tag-resources --resource-arn-list \
    arn:aws:lambda:us-west-1:133577413914:function:Production-audits-0-SQS \
    arn:aws:lambda:us-west-1:133577413914:function:Production-product-0-InitDB \
    arn:aws:lambda:us-west-1:133577413914:function:Production-GithubActionWorkflowBuilderGithubOAuthCodeProxy \
    arn:aws:lambda:us-west-1:133577413914:function:Production-audits-0-Web \
    arn:aws:lambda:us-west-1:133577413914:function:Production-audits-0-InitDB \
    --tags Environment=Development \
    --region us-west-1 
```

在 [AWS 文档](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-existing-stack.html)中描述了将非托管资源导入 CloudFormation 堆栈的过程。

## 结论

确定 AWS 客户中的影子 IT 资源是建立可由开发运维团队有效管理的基础架构的第一步。然后，通过建立一致的标记方案，您能够记录谁负责什么、资源的用途以及哪些外部过程创建了它们。

在这篇文章中，您看到了许多用于查找不兼容资源的脚本，以及关于如何添加缺失标签或将资源导入 CloudFormation 堆栈的提示。随着基础架构的规模和复杂性的增长，这些简单的步骤可以带来巨大的变化。

阅读我们的 [Runbooks 系列](https://octopus.com/blog/tag/Runbooks%20Series)的其余部分。

愉快的部署！