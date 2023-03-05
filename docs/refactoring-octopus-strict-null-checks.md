# 重构 Octopus:向 Octopus 前端添加严格的空检查——Octopus Deploy

> 原文：<https://octopus.com/blog/refactoring-octopus-strict-null-checks>

[![Refactoring Octopus: Adding strict null checks to the Octopus front-end](img/7d39b7ac84db263484736c1fdbb70fc6.png)](#)

空值通常被认为是十亿美元的错误。它们会在你最意想不到的时候悄悄出现，而且以一些非常有趣的方式出现。问题不在于 null 的表示本身，更多的是我们经常忘记处理`null`的情况。正是因为这个原因，一些语言完全回避 null 的概念，而选择用类型安全的方式使用`Option`单子来表示这个概念。函数式编程并不总是一件容易的事情，在某些情况下，我们可能仍然有遗留的代码库。这就是`strictNullChecks`编译器标志介入并拯救世界的地方。这个单独的开关允许 TypeScript 将`null`和`undefined`作为单独的类型，强制处理这些情况。这减少了围绕`null`和`undefined`的错误，并在适当缩小类型时消除了很多复杂性。它还消除了为某些情况编写测试的需要，并提供了更快的反馈。如果这引起了您的注意，让我们看看在现有代码库中启用严格空值的一些策略。

## 可能的选择

在现有的代码库上启用严格的空值应该像按下开关一样简单，对吗？不幸的是，事情没那么简单。在我们的例子中，仅仅启用这个标志就导致了大约 5000 个错误，这不是一个小数目。这意味着我们必须确定启用严格空值的最佳行动方案。让我们来看几个选项:

### 一气呵成

在许多情况下，您可以一次转换所有内容。也许你已经编写了完全严格的 null 兼容代码，这似乎不太可能，或者也许你有一个非常小的代码库。在我们的特殊情况下，除了未知因素之外，我们还必须做出巨大的改变，这使我们无法成功。这听起来可能需要一些艰苦的工作才能实现，但是，我们发现，除非您应用外科手术式的精确打字更改，否则您很可能会为您所做的每个打字更改引入更多的错误。当数字在你下面不停变化的时候，不容易估计。

### 隔离我们的前端代码库

随着 monorepos 和 TypeScript 支持项目引用的流行，这是一个相当有吸引力的选择。这将有效地允许我们获取代码库的片段，并将它们分离到单独的项目中，允许我们为每个单独的片段递增地启用严格的空值。乍一看，这个选项非常吸引人，但是围绕它的工具还不太完整。围绕`fork-ts-checker-webpack-plugin`中的项目引用支持有许多问题，而项目引用目前对于基于 react 的库来说没有多大意义。这仍然是我们希望在未来重新审视的东西，但这不是我们想要转换成严格空值的东西。

### 多个 tsconfigs

我们可以引入多个`tsconfig.json`文件，一个排除严格空值，另一个包含严格空值。在开发过程中，我们可以启用严格空值，同时使用不同的配置来排除生产构建的严格空值。这将让我们随着时间的推移对不兼容的代码施加向下的压力，但这也意味着您可能会有 7000 多个错误。这感觉不太理想。对此的一种变体是让 IDE 使用启用了严格空值的 tsconfig，这将所有错误排除在控制台之外。

### 带有断言的多个 tsconfigs

使用这种策略，您可以为正常的开发和生产构建禁用严格的空值。然后，您将添加一个启用了严格空值的附加 tsconfig，它包含一个明确的兼容文件列表。这还需要在 CI 构建中增加一个步骤，对包含严格空检查标志的 tsconfig 运行`tsc`检查。这确保了引入的额外错误会被发现，但是在正常开发过程中您可能不一定会看到这些错误。当有意地在较短的时间内为这一努力贡献重要的工程努力时，这可能会工作得很好。

### 非空断言运算符

TypeScript 有一个非空断言操作符(`!`)，它作为一种手段告诉 TypeScript 在特定的范围内某些东西不是空的。这允许围绕`null`和`undefined`的打字问题在逐个情况的基础上被抑制，但是当严格空值被启用时，它要求对每个错误进行抑制。启用林挺规则来阻止操作符的使用，并逐个文件地抑制这些规则，也有利于防止操作符的传播。这种方法允许您在文件级别有效地启用严格的空值，并且可以预计到相关的工作量。

我们决定将此作为首选，因为我们希望以一种可预测的方式实现严格的空值，并且风险和影响尽可能小。

### 关于这种方法的警告

所选择的方法有一些突出的缺点，非常值得一提:

*   它需要大量的机械抑制。
*   它可能导致部分实现的严格的空代码库，如果没有集中精力，它永远不会完全完成。
*   如果不对非空断言进行控制，它们就会传播开来。
*   以这种方式使用非空断言操作符，可能会产生一些令人讨厌的代码味道，例如:

```
 this.state = {
        variables: null!;
    } 
```

我们也知道，从长远来看，我们需要付出更多的努力来解决这些问题，所以我们认为在我们的情况下，利大于弊。

## 经验教训

作为我们转换过程的一部分，在到处应用非空断言之后，我们没有让事情保持原样。我们还想把我们的字体端的一个特征区域转换成严格的 null 兼容。我们知道，为了让我们的生活变得更轻松，我们需要更好地理解我们需要做出的改变以及应用的模式。以下是我们在这个过程中吸取的一些教训。

### 类型的交集可以是互斥的

如前所述，启用了严格空值的 TypeScript 将把`null`和`undefined`视为不同的类型，并将对类型的交集应用更严格的检查。一个例子是使用一个`Record`和另一个包含可选键属性的类型之间的交集。以下面这个为例:

```
type RouteArgs = Record<string, string>;
type AllArgs = { ids?: string [] };
type SomeArgs = RouteArgs & AllArgs;
const args: SomeArgs = { ids: undefined }; //invalid 
```

在启用严格空值之前，这种事情本来是可行的，但是之后，您会看到一条消息，指出属性 id 与索引签名不兼容，并且不能分配给`undefined`。如果你看看`Record`的签名，这就完全说得通了:

```
type Record<K extends keyof any, T> = {
    [P in K]: T;
}; 
```

这意味着我们不能提供`undefined`作为`Record`的键，因为`keyof` `undefined`评估为底部类型`never`。这也意味着类型之间的交集没有意义，导致可选的 prop 被丢弃，结果是 TypeScript 抱怨。值得注意的是，随着 [TypeScript 3.9](https://devblogs.microsoft.com/typescript/announcing-typescript-3-9-rc/#breaking-changes) 的发布，TypeScript 对这类事情越来越严格。

### 在混合使用未定义的情况下，对类型进行索引会变得很棘手

假设您有一个带有深度嵌套状态的 React 组件，并且您希望对它的一小部分进行某种类型安全的变更，同时避免安装新的依赖项(如 immer)。我们称这个方法为 setChildState。其用法的一个虚构示例如下所示:

```
 type Address = {
        details: string;
        postcode: string;
    };

    type Person = {
        name: string;
        surname: string;
        address: Address;
    };

    type State = {
        person: Person;
    };

    class MyComponent extends React.Component<{}, State>{
        constructor(props: Props){
            super(props);
            this.state = {
                //..state
            }
        }
        setChildState<KeyOfState extends keyof State, Child extends State[KeyOfState], KeyOfChild extends keyof Child, GrandChild extends Child[KeyOfChild], KeyOfGrandChild extends keyof GrandChild>(
            first: KeyOfState,
            second: KeyOfChild,
            state: Pick<GrandChild, KeyOfGrandChild>,
            callback?: () => void
        ) {
            this.setState(
                prev => ({
                    [first]: {
                        ...prev[first],
                        [second]: {
                            ...(prev[first] as Child)[second],
                            ...state,
                        },
                    },
                }),
                callback
            );
        }

        changeAddress(address: Address) {
            this.setChildState("person", "address", address);
        }

        render() {
            return (
                <div>
                    <button
                    onClick={() =>
                        this.changeAddress({ postcode: "555", details: "Nowhere" })
                    }
                    >
                    Change Address
                    </button>
                </div>
            );
        }
    } 
```

不幸的是，在您启用了严格的空值之后，这种情况就不太好了，并且您在链的下游有一个 null/undefined。你可以用这个例子自己尝试一下。让`State`中的`person`成为可选的将会破坏事物，并且有很好的理由。如果该属性是可选的，那么您可能正在传播一个`undefined`对象，这将使状态处于某种模糊状态，即只有对象的一部分被定义。在这一点上，你实际上是在撒谎什么是定义了的，什么是没有定义的。当然，如果你不在乎，你可以有策略地在整个签名中散布`NonNullable`，但是你最好通过使用道具来传递值，从而将`null`和`undefined`完全排除在外。稍后我们会看到一个这样的例子。

### 严格的空值也意味着函数周围更严格的类型

TypeScript 允许您启用其他严格的标志，如`strictFunctionTypes`，这将启用更严格的双变量检查。然而，您可能会惊讶地发现，当仅启用`strictNullChecks`时，一些更严格的检查可能会应用于与`null`和`undefined`无关的功能。React `refs`就是一个很好的例子。你可能以前尝试过使用一些基本类型，如`HTMLElement`作为你的引用，如下所示:

```
export default function App() {
  const myRef = React.useRef<HTMLElement | undefined>();

  return (
    <div ref={myRef} className="App"></div>
  );
} 
```

TypeScript 允许您直接将赋值给基类型，但对于函数来说就不那么宽松了，因此您可能需要缩小类型的范围:

```
export default function App() {
  const myRef = React.useRef<HTMLDivElement | undefined>();
//Okay
  return (
    <div ref={myRef} className="App"></div>
  );
} 
```

很快，类型错误消失了，编译器很高兴。

### 先说资源差异

还记得我们说过严格空值将`null`和`undefined`视为不同的类型吗？在我们的特殊情况下，这也意味着我们必须为新的和现有的资源稍微改变接口。这是因为创建资源所需的属性可能比指定的少。当然，TypeScript 并不知道这一点，它只是想有所帮助。让我们以下面的资源为例:

```
interface TenantLinks {
    Self: string;
    Variables: string;
    Web: string;
    Logo: string;
}

export interface TenantResource {
    Id: string;
    Name: string;
    TenantTags: string[];
    ProjectEnvironments: { [projectId: string]: string[] };
    ClonedFromTenantId: string | null;
    Description: string | null;
    SpaceId: string;
    Links: LinksCollection<TenantLinks>;
} 
```

启用严格的空值后，每当我们第一次创建资源时，我们都需要指定链接、Id、ClonedFromTenantId 以及空间 ID，尽管这些对于新资源都没有意义。这是因为相关的链接是由服务器提供的，我们自动生成 ID，自动推断空间 ID，ClonedFromTenantId 由服务器在克隆租户时填充。我们选择将这些资源的共享属性提取到一个单独的接口中，并根据需要更改属性。

这最终看起来像这样:

```
interface TenantLinks {
    Self: string;
    Variables: string;
    Web: string;
    Logo: string;
}

interface TenantResourceShared {
    TenantTags: string[];
    ProjectEnvironments: { [projectId: string]: string[] };
    Name: string;
}

export interface TenantResource extends TenantResourceShared {
    Id: string;
    SpaceId: string;
    ClonedFromTenantId: string | null;
    Description: string | null;
    Links: LinksCollection<TenantLinks>;
}

export interface NewTenantResource extends TenantResourceShared {
    Description?: string;
} 
```

随着这一变化，我们的存储库接受了`NewResource`,同时总是返回完整的资源以供使用。同样，任何`fetch`操作都将返回完整的资源。

### 尽快缩小一次

如何创建类型来表示事物可能相当主观。您可以使用 instanceof 创建类，并基于构造函数进行继承和区分。您还可以创建对象，并通过类似于 redux 操作的属性进行区分。区别工会的优势在于，他们不一定有完全相同的形状。在这两种情况下，我们都需要缩小类型的范围来决定要做什么。这同样适用于原语，其中`null`和`undefined`作为类型集的一部分，例如`string | null | undefined`。我们遵循一个简单的规则来避免到处添加可选的链接和/或空检查。为了实现这一点，我们在尝试使用一个类型之前，尽可能地缩小它的范围。

以上是一个简单的规则，可以大大降低复杂性。这感觉很明显，但原因是因为它减少了您需要处理的排列数量。字体越宽，你需要处理的案例就越多，出错的几率就越大。最好尽快缩小到最严格的类型。例如，我们更喜欢`string`而不是`string | null`，而对于更复杂的对象和类层次，我们更喜欢可能的更具体的东西。

### 加载数据

在使用 react 时，在`componentDidMount`或钩子中加载数据是一件很常见的事情。问题是您可能没有可用于第一次渲染的数据，这需要将数据标记为可选的。

一种解决方案是通过 props 传递数据，并且只在知道有可用数据时才呈现组件。这种模式变得如此普遍，以至于我们创建了一个组件，专门加载我们的数据，并通过 props 注入数据，巧合的是，这也遵循了我们的尽快缩小范围的规则。这最终看起来像下面这个简化的例子。如果您以前使用过 GraphQL Apollo 客户端，这可能看起来有些熟悉(如果您足够仔细地观察):

```
const Loader = DataLoader<PersonResource>();

export const ExamplePage = () => {
  return (
    <Loader
      load={async () => {
        return await getPerson("Person-1");
      }}
      renderWhenLoaded={data => {
        return <PersonLayout initialData={data} />;
      }}
      renderAlternate={p => <div>Loading</div>}
    />
  );
}; 
```

### 默认

您可能会发现，可以对某些类型使用默认的大小写，而不是将现有的类型修改为可选的、`undefined`或`null`。例如，您可以:

*   默认数字为`0`。
*   默认字符串为`""`。
*   默认数组为`[]`。
*   默认布尔值为`false`。
*   将所有可选属性默认为`{}`的对象。
*   定义为`{ [key: string]: T }`到`{}`的默认对象。

这并不总是可能的，尤其是在涉及数组或者代码特别寻找`null`或`undefined`的时候，所以首先检查一下是值得的。

### 可选参数，空或未定义

如果`null`和`undefined`被视为不同的类型，并且我们有办法指定可选的道具，那么我们如何决定我们是否应该使用像`{ name: string | undefined }`、`{name: string | null }`、`{ name?: string }`甚至`{ name: string | null | undefined }`这样的类型签名呢？这很可能归结为意图，以及你是否希望消费者被迫考虑他们提供的东西。出于这个原因，我们尽量避免可选道具，倾向于在类型中添加`null`。

根据我们的经验，可选道具被滥用时，会导致一些微妙的错误，并且很难发现不正确的用法。如果可能的话，通过默认一个可选参数来防止它被广泛传播通常也是一个好主意。

## 结论

在开始一个新项目时，从最严格的 TypeScript 编译器规则开始绝对是最好的选择。如果你没有考虑，请考虑。你未来的自己会感谢你。如果你没有那么幸运，并且你有一个很大的现有代码库，可能需要一些认真的、专注的努力才能实现。似乎没有一个选项是完美的，所以最好选择最适合您的特定场景的选项，然而，这种努力似乎是值得的。我们讲述了在转换 Octopus 中的某个特定区域使其严格符合 nulls 时学到的一些东西。该列表绝非详尽无遗，随着我们继续转换剩余区域的旅程，我们希望了解更多信息。我们很想听听你关于你用来处理严格空值的模式，以及是否有我们在这篇文章中没有提到的其他问题和经验。

下次再见，愉快的部署！