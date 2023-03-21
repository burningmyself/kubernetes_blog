
## 流程控制

> 控制流程为模板作者提供了控制模板生成流程的功能，Helm 的模板语言提供了以下一些流程控制：

*   `if/else` 条件语句
*   `with` 指定一个作用域范围
*   `range` 提供类似于 `for each` 这样的循环样式

> 除此之外，还提供了一些声明和使用命名模板的操作：

*   `define` 在模板内部声明一个新的命名模板
*   `template` 导入一个命名模板
*   `block` 声明了一种特殊的可填充模板区域。

> 这里我们先来了解 `if`、`with`、`range` 语句的使用，其他将在后面的`命名模板`部分介绍。

### if/else

> 首先我们先来了解下有条件地在模板中包含一个文本区域，就是 `if/else` ，这个条件判断的基本结构如下所示：

```
{{ if PIPELINE }}
  # Do something
{{ else if OTHER PIPELINE }}
  # Do something else
{{ else }}
  # Default case
{{ end }}
```

> 可以看到我们这里判断的是管道而不是一个 values 值，这是因为控制结构可以执行整个管道，而不仅仅是判断值。如果值为以下的一些内容，则将管道判断为 false：

*   布尔 false
*   数字零
*   一个空字符串
*   nil（empty 或者 null）
*   一个空集合（map、slice、tuple、dict、array）

> 在其他条件下，条件都为真。

> 现在我们在上面的示例模板 ConfigMap 中添加一个简单的条件，如果 drink 设置为 coffee，我们就添加另外一个设置：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}mug: true{{ end }}
```

> 我们把 values.yaml 文件内容设置成下面的样子：

```
favorite:
  # drink: coffee
  food: pizza
```

> 由于我们注释掉了 `drink: coffee`，所以渲染后输出不会包含 `mug: true` 的标志，但是如果我们把注释取消掉，则应该输出如下所示的内容：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1575970308-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: true
```

> 这是因为上面模板中我们添加了 

```
if eq .Values.favorite.drink "coffee"
```

 这样的条件判断，相当于是判断 

```
.Values.favorite.drink
```

 值是否等于 `"coffee"`，如果相等则渲染 `mug: true`。

### 空格控制

> 还有一个非常重要的功能点就是关于空格的控制，因为空格对于 YAML 文件非常重要的，不是说任意缩进就可以，依然还是以前面的例子为例，我们来格式化下模板格式以更易于阅读：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}
    mug: true
  {{ end }}
```

> 现在我们的模板看上去更易于阅读了，但是我们通过模板引擎来渲染下，却会得到如下的错误信息：

```
$ helm install --generate-name --dry-run --debug ./mychart
install.go:148: [debug] Original chart version: ""
install.go:165: [debug] CHART PATH: /Users/ych/devs/workspace/yidianzhishi/course/k8strain/content/helm/manifests/mychart

Error: YAML parse error on mychart/templates/configmap.yaml: error converting YAML to JSON: yaml: line 9: did not find expected key
```

> 这是因为我们在模板中添加了空格，生成了不正确的 YAML 文件：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1575970308-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
    mug: true
```

> 我们可以看到 `mug: true` 的缩进是有问题的，不符合 YAML 文件格式，现在我们讲缩进去掉试看看：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}
  mug: true
  {{ end }}
```

> 重新渲染模板，然后可以发现已经可以正常通过了，但是渲染出来的 YAML 文件格式看上去还是有点奇怪：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1575971172-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"

  mug: true
```

> 我们可以看到得到的 YAML 文件中多了一些空行，这是因为模板引擎渲染的时候它会删除 `{{` 和 `}}` 之间的内容，但是会完全保留其余的空格。我们知道在 YAML 文件中空格是有意义的，所以管理空格就变得非常重要了，不过 Helm 模板也提供了一些工具来帮助我们管理空格。

> 首先可以使用特殊字符修改模板声明的花括号语法，以告诉模板引擎去掉空格。`{{-` 添加了破折号和空格表示应将左边的空格移除，`-}}`表示将右边的空格移除，`另外也需要注意的是，换行符也是空格`。

> 空格

> 需要注意的时候要确保 `-` 和指令的其余部分之间要有空格，`{{- 3 }}` 表示删除左边的空格并打印3，但是 `{{-3 }}`表示打印`-3`。

> 使用这个语法，我们可以修改上面的模板来移除多余的空行：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee" }}
  mug: true
  {{- end }}
```

> 渲染后可以看到空行被移除掉了：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1575972373-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: true
```

> 为了更加清楚地说明这个问题，我们用`*`来代替将要删除的每个空格，行尾的`*`表示被删除的换行符：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}*
**{{- if eq .Values.favorite.drink "coffee" }}
  mug: true*
**{{- end }}
```

> 所以我们这里用 `{{-` 表示的就是删除本行开头的两个空格以及上一行的换行符，这样是不是就将空行都删除了啊。

> 在使用移除空格的时候还需要小心，比如下面的操作：

```
food: {{ .Values.favorite.food | upper | quote }}
{{- if eq .Values.favorite.drink "coffee" -}}
mug: true
{{- end -}}
```

> 我们依然还是可以用 `*` 来代替空格进行分析，如下所示：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}*
**{{- if eq .Values.favorite.drink "coffee" -}}*
  mug: true*
**{{- end -}}
```

> 第一个 `{{-` 会删除前面的空格和前面的换行符，然后后面的 `-}}` 会删除当前行的换行符，这样就会把 `mug: true` 移动到 `food: "PIZZA"` 后面去了，最终渲染过后就会变成：

```
food: "PIZZA"mug: true
```

，因为在两侧都去掉换行符。

> > 有关模板中空格控制的详细信息，可以查看 Go 模板官方文档介绍。

> 不过有时候告诉模板系统如何缩进比起去控制模板指令的间距更加容易，所以，有时候你会发现缩进函数（

```
{{ indent 2 "mug: true" }}
```

）更有用。

### 使用 with 修改作用域

> 接下来需要了解的是 `with` 操作，它可以控制变量的作用域，然后重新用 `.` 调用就表示对当前作用域的引用，所以，`.Values` 是告诉模板引擎在当前作用域下内查找 Values 对象。

> `with` 语句的语法和 `if` 语句比较类似：

```
{{ with PIPELINE }}
  # 限制范围
{{ end }}
```

> 范围可以更改，可以让你将当前范围 `.` 设置为特定的对象，例如，我们一直在使用 `.Values.favorites`，让我们重写下模板文件 ConfigMap 来更改 `.` 的范围指向 `.Values.favorites`：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
```

> 我们这里将前面练习的 `if` 条件语句删除了，在模板中我们添加了一个 

```
{{- with .Values.favorite }}
```

 的语句，意思就是说在 `with` 语句的作用范围内可以用 `.` 表示 `.Values.favorite` 了，所以我们可以引用 `.drink` 和 `.food` 了，但是在 `{{ end }}` 之后就会重置为之前的作用域了。

> 不过需要注意得是，在受限的作用域范围内，你无法从父级范围访问到其他对象，比如，下面得模板会失败：

```
{{- with .Values.favorite }}
drink: {{ .drink | default "tea" | quote }}
food: {{ .food | upper | quote }}
release: {{ .Release.Name }}
{{- end }}
```

> 因为 `Release.Name` 并不在 `.` 的限制范围内，所以会产生错误，但是，如果我们交换最后两行，则就可以正常工作了，因为 `{{ end }}` 之后会重置作用域。

```
{{- with .Values.favorite }}
drink: {{ .drink | default "tea" | quote }}
food: {{ .food | upper | quote }}
{{- end }}
release: {{ .Release.Name }}
```

> 下面我先来了解下 `range`，然后我们再去了解下模板变量，它可以为上面得这个范围问题提供一种解决方案。

### range 循环操作

> 我们知道许多编程语言都支持使用 `for` 循环、`foreach` 循环或者类似功能机制进行循环迭代，在 Helm 得模板语言中，迭代集合得方法是使用 `range` 运算符。

> 比如首先我们在 `values.yaml` 文件中添加一份 pizza 馅料列表：

```
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

> 现在我们有了 `pizzaToppings` 列表（在模板中称为切片），我们可以来修改下模板将列表打印到 ConfigMap 中：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  toppings: |-
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}
```

> 我们仔细观察下模板中的 `toppings:` 列表，`range` 函数将遍历 `Values.pizzaToppings` 列表，我们看到里面使用了一个 `.`，类似于上面我们用 `with` 设置范围一样，运算符也是这样的，每次循环，`.` 都会被设置为当前的 `pizzaTopping`，也就是说第一次设置为`mushrooms`，第二次迭代设置为`cheese`，依次类推。

> 我们可以直接传递 `.` 这个值到管道上，所以我们这里 

```
{{ . | title | quote }}
```

 就相当于发送 `.` 给 `title`（标题大小写函数）函数，然后发送给 `quote` 函数，我们渲染这个模板，会输出如下的内容：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1575975849-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  toppings: |-
    - "Mushrooms"
    - "Cheese"
    - "Peppers"
    - "Onions"
```

> 在上面模板中，我们做了一些小小的特殊处理，`toppings: |-` 行表示声明一个多行字符串，所以其实我们的 `toppings` 列表不是一个 YAML 列表，而是一个比较大的字符串，这是因为 ConfigMap 中的数据由 `key/value` 对组成，所有 key 和 value 都是简单的字符串，要了解为什么是这样的，可以查看 Kubernetes ConfigMap 文档，不过这个细节对我们这里不重要。

> YAML

> 多行字符串可以使用 `|` 保留换行符，也可以使用 `>` 折叠换行，如：

```
this: |
 Foo
 Bar
that: >
 Foo
 Bar
```

> 对应的意思就是：

```
{this: Foo\nBar\n, that: Foo Bar\n}
```

> `+` 表示保留文字块末尾的换行，`-` 表示删除字符串末尾的换行，如：

```
s1: |
Foo

s2: |+
Foo

s3: |-
Foo
```

> 对应的意思就是：

```
{s1: Foo\n, s2: Foo\n\n\n, s3: Foo}
```

> 有时候，在模板中快速创建一个列表，然后遍历该列表很有用，Helm 模板具有简化该功能的函数：`tuple`。元组是固定大小的列表集合，但是具有任意数据类型，下面是元组的大概使用方法：

```
sizes: |-
  {{- range tuple "small" "medium" "large" }}
  - {{ . }}
  {{- end }}
```

> 上面的模板最终会被渲染成如下的 YAML：

```
sizes: |-
  - small
  - medium
  - large
```

> 除了列表和元组之外，`range` 还可以用于遍历字典，我们在下一节介绍模板变量的时候再来了解这个用法吧。