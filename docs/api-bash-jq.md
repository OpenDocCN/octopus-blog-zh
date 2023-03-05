# 通过 Bash 和 jq - Octopus Deploy 使用 Octopus API

> 原文：<https://octopus.com/blog/api-bash-jq>

Octopus Deploy 是 API 优先编写的，这意味着您可以在用户界面(UI)中做的任何事情，都可以通过 API 调用来完成。当我与 API 交互时，我使用 PowerShell，因为它具有将 JSON 转换为 PowerShell 对象的内置功能，这使得使用 JSON 很容易。然而，并不是所有的 Octopus 客户都使用 PowerShell，有些客户需要使用 Bash 的基于*nix 的解决方案。

在这篇文章中，我演示了如何使用 Bash 和 Octopus API。

## japan quarterly 日本季刊

Octopus Deploy API 以 JSON 格式返回数据。然而，Bash 没有提供处理 JSON 数据的内置方法，而是将 JSON 视为字符串。虽然 Bash 有一些复杂的字符串操作功能，但是使用 JSON 数据仍然很困难。

为了解决这个问题，Linux 社区开发了一个强大的命令行实用程序 jq 来解析 JSON。Jq 已经成为处理 JSON 数据的首选工具，可以通过使用包管理器(如 APT 或 YUM)轻松安装。

## 卷曲

Wget 和 cURL 是使用 Bash 发出 web 请求的两种最常用的方法。有这么多可用的 Linux 发行版，您无法判断安装的是哪一个实用程序。本文中的例子使用了 cURL 工具。像 jq 一样，cURL 可以使用包管理器来安装。

## 使用分页数据

Octopus 中的许多 API 以分页格式返回数据。对于这些 API 调用，您可以提供 querystring 参数如`skip`和`take`来操纵调用的结果。请参见 https://[YourServer]/api，获取 api 方法列表和每种方法的 querystring 参数列表。

有时您希望返回给定调用的所有结果，例如当您检索所有项目时。对于这种情况，我编写了一个 Bash 函数，递归调用 API，直到返回所有结果。

虽然这篇文章的这一部分谈到了分页数据，但是`Get-OctopusItems`函数也适用于非分页 API 调用。

```
Get-OctopusItems () {
    octopusUri=$1
    apiKey=$2
    skipCount=$3

    header="X-Octopus-Apikey: $apiKey"
    items=()
    skipItemQuerystring=""

    # Adjust querysting accordingly
    if [[ "$octopusUri" == *"?"* ]]; then
        skipItemQuerystring="&skip="
    else
        skipItemQuerystring="?skip="
    fi

    # Append the amount to skip
    skipItemQuerystring+="$skipCount"

    # Get results
    resultSet=$(curl -H "$header" -H "Accept: application/json" -H "Content-Type: application/json" "$octopusUri$skipItemQuerystring")

    # Check to see if items is present
    items=$(echo "$resultSet" | jq .Items)

    if [[ ! -z "$items" ]]; then

        # Check to see if results are bigger than page count
        itemsPerPage=$(echo "$resultSet" | jq -r .ItemsPerPage)
        totalResults=$(echo "$resultSet" | jq -r .TotalResults)
        itemCount=$(echo "$(($totalResults - $skipCount))")

        if [[ "$itemCount" -gt "$itemsPerPage" ]]; then

            # Increment skip count
            skipCount=$(echo "$(($itemsPerPage + $skipCount))")

            # Recursively call
            items+=$(Get-OctopusItems "$octopusUri" "$apiKey" "$skipCount")
        fi
    else
        echo "$items"
    fi

    echo "$items"
}

OctopusUrl="https://yourserverurl"
spaceId="Spaces-1"
ApiKey="API-YourAPIKey"

# Get all the projects
projects=$(Get-OctopusItems "$OctopusUrl/api/$spaceId/projects" "$ApiKey" 0)

if [[ "$projects" == *"]["* ]]; then
    # Replace characters to make one contiguous array
    projects=$(echo "${projects//][/,}")
fi 
```

对 API 的每个调用都返回一个项目数组。返回的 JSON 中有多个 JSON 数组。

`if`语句通过测试`][`的字符串并在找到时用`,`替换它来检查是否返回了许多数组。这使得 JSON 字符串成为单个数组，更容易处理。

在返回所有项目数据之后，您可以使用更多的 jq 命令来检索元素，比如 ProjectId，并检索项目特定的数据，比如部署过程、操作手册或变量。

```
# Iterate over returned items
arr=( $(echo "$projects" | jq -r '.[].Id') )

for projectId in "${arr[@]}"
do
    echo "$projectId"
done 
```

## 用 Bash 发布到 API

上面的例子演示了如何向 API 发出 GET 请求。虽然有用，但是这些请求只是您与 API 交互的一部分。下面的示例创建一个 JSON 文档，以发送到中断 API:

```
automaticResponseReasonNotes="Because I said so!"
automaticResponseManualInterventionResponseType="Proceed"

# Create JSON document
jsonBody=$(jq -n \
        --arg notes "$automaticResponseReasonNotes" \
        --arg result "$automaticResponseManualInterventionResponseType" \
        '{"Notes": $notes, "Result": $result}' )

# Submit response
curl -H "$header" -H 'Content-Type: application/json' -X POST -d "$jsonBody" "$automaticResponseOctopusUrl/api/$spaceId/interruptions/$manualInterventionId/submit" 
```

## 结论

这篇文章演示了如何使用 Bash、cURL 和 jq 与 Octopus API 进行交互。 [Octopus API 示例](https://octopus.com/docs/octopus-rest-api/examples)页面包含更多在 PowerShell、C#、Python 和 Go 中使用 API 的示例。

愉快的部署！