
## 函数和管道

> 现在我们已经了解了如何将信息加入到模板中，但是这些信息都是直接原样的放置过去的，有时候，我们希望以一种对我们更有用的方式来转换提供的数据。

> 下面让我们从一个最佳实践开始：将 `.Values` 对象中的字符串注入模板时，我们应该引用这些字符串，我们可以通过在 template 指令中调用 `quote` 函数来实现，比如：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```

> 模板函数遵循的语法规则是 

```
functionName arg1 arg..
```

，在上面的代码片段中，

```
quote .Values.favorite.drink
```

 会调用 `quote` 函数并传递一个单个参数。

> Helm 有60多种可用的函数，其中一些是由 Go 模板语言本身定义的，其他大多数都是 Sprig 模板库提供的，接下来我们会通过部分示例来逐步介绍其中的一些功能函数。

> Helm 模板

> 当我们谈论 `Helm 模板语言` 的时候，就好像是特定于 Helm 一样，但实际上它是 Go 模板语言加上一些额外的函数以及各种封装程序的组合，以将某些对象暴露给模板。当我们需要学习模板的时候，Go 模板上有许多资源会对我们有所帮助的。

### 管道

> 模板语言有一个强大的功能就是`管道（Pipeline）`概念，管道利用 UNIX 的概念，将一系列模板命令链接在一起，一起对外提供服务，换句话说，管道是按顺序完成多项工作的有效方式，我们来使用管道重写上面的示例模板：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | quote }}
```

> 在这里我们没有调用 `quote ARGUMENT` 函数，而是颠倒了下顺序，我们使用管道符（|）将参数`发送`给函数：

```
.Values.favorite.drink | quote
```

，使用管道，我们可以将多个功能链接在一起：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

> 管道顺序

> 反转顺序是模板中常见的做法，我们会看到 `.val | quote` 比 `quote .val` 用法更多，虽然两种方法都是可以的。

> 最后，模板渲染后，会产生如下所示的结果：

```
$ helm install --generate-name --dry-run --debug ./mychart
install.go:148: [debug] Original chart version: ""
install.go:165: [debug] CHART PATH: /Users/ych/devs/workspace/yidianzhishi/course/k8strain/content/helm/manifests/mychart

NAME: mychart-1575966483
LAST DEPLOYED: Tue Dec 10 16:28:04 2019
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
favorite:
  drink: coffee
  food: pizza
favoriteDrink: coffee

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1575966483-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

> 我们可以看到 values 中的 `pizza` 值已经被转换成了 `"PIZZA"`。当这样传递参数的时候，第一个求值结果（

```
.Values.favorite.drink
```

）会作为一个参数发送给函数，我们可以修改上面的 `drink` 示例，用一个带有两个参数的函数进行说明：`repeat COUNT STRING`。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | repeat 5 | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

> `repeat` 函数将重复字符串给定的次数，渲染后我们可以得到如下的输出结果：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1575966939-configmap
data:
  myvalue: "Hello World"
  drink: "coffeecoffeecoffeecoffeecoffee"
  food: "PIZZA"
```

### `default` 函数

> 在模板中经常会使用到的一个函数是 `default` 函数：

```
default DEFAULT_VALUE GIVEN_VALUE
```

，该函数允许你在模板内部指定默认值，我们来修改上面示例中的模板：

```
food: {{ .Values.favorite.food | default "rice" | upper | quote }}
```

> 正常运行，我们还是可以得到 `values.yaml` 文件中定义的 pizza：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1575966939-configmap
data:
  myvalue: "Hello World"
  drink: "coffeecoffeecoffeecoffeecoffee"
  food: "PIZZA"
```

> 现在我们从 `values.yaml` 文件中移除 food 的定义：

```
favorite:
  drink: coffee
  # food: pizza
```

> 现在我们重新运行 

```
helm install --generate-name --dry-run --debug ./mychart
```

 将渲染成如下的 YAML 文件：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1575967394-configmap
data:
  myvalue: "Hello World"
  drink: "coffeecoffeecoffeecoffeecoffee"
  food: "RICE"
```

> 在一个真实的 chart 模板中，所有的静态默认值都应位于 `values.yaml` 文件中，并且不应该重复使用 `default` 函数，但是，默认命令非常适合计算不能在 `values.yaml` 文件中声明的 values 值，例如：

```
food: {{ .Values.favorite.food | default (printf "%s-rice" (include "fullname" .)) }}
```

> 不过在有些地方，`if` 条件语句可能比 default 函数更合适，我们会在后面了解到。

> 模板函数和管道是将数据转换后然后将其插入到 YAML 文件中的一种强大方法，但是有的时候有必要添加一些模板逻辑，这些逻辑比仅仅插入字符串要复杂得多，下面我们将来了解模板语言中提供的控制流程。

### 运算符函数

> 另外需要注意的是在模板中，运算符（eq、ne、lt、gt、and、or 等等）均实现为函数，在管道中，运算符可以用括号`（）`进行分割。

> 接下来我们可以去了解控制流程条件语句、循环和作用域修饰符的使用。