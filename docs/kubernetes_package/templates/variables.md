
## 变量

> 有了函数、管道、对象以及控制结构，我们可以想象下大多数编程语言中更基本的思想之一：`变量`。在模板中，变量的使用频率较低，但是，我们还是可以使用他们来简化代码，以及更好地使用 `with` 和 `range`。

> 在前面的示例中，我们知道下面的模板渲染会出错：

```
{{- with .Values.favorite }}
drink: {{ .drink | default "tea" | quote }}
food: {{ .food | upper | quote }}
release: {{ .Release.Name }}
{{- end }}
```

> 因为 `Release.Name` 不在 `with` 语句块限制的范围之内，解决作用域问题的一种方法是将对象分配给在不考虑当前作用域情况下访问的变量。

> 在 Helm 模板中，变量是对另外一个对象的命名引用。它遵循 `$name` 格式，变量使用特殊的赋值运算符进行赋值 `:=`，我们可以修改上面的模板，为 `Release.Name` 声明一个变量：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- $relname := .Release.Name -}}
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ $relname }}
  {{- end }}
```

> 注意在 `with` 语句之前，我们先分配了 

```
$relname := .Release.Name
```

，然后在 `with` 语句块中，`$relname` 变量仍然表示 release 的名称，我们渲染该模板，可以得到如下的正确结果：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1575982655-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  release: mychart-1575982655
```

> 变量在 `range` 循环里面非常有用，它们可以用于类似于列表的对象来捕获索引和 value 值：

```
toppings: |-
  {{- range $index, $topping := .Values.pizzaToppings }}
    {{ $index }}: {{ $topping }}
  {{- end }}
```

> 注意 `range` 在前面，然后是变量，然后是赋值运算符，然后才是列表，这会将整数索引（从0开始）分配给 `$index`，并将 value 值分配给 `$topping`，上面的内容会被渲染成如下内容：

```
toppings: |-
  0: mushrooms
  1: cheese
  2: peppers
  3: onions
```

> 对于同时具有 key 和 value 的数据结构，我们也可以使用 `range` 来获得 key、value 的值，比如，我们可以像这样循环遍历 `.Values.favorite`：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

> 在第一次迭代中，`$key` 是 `drink`，`$val` 是 `coffee`，在第二次迭代中，`$key` 是 `food`，`$val` 是 `pizza`。运行上面的命令将生成下面的内容：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1575983119-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

> 一般来说变量不是全局的，它们的作用域是声明它们的块区域，之前，我们在模板的顶层分配了 `$relname`，该变量将在整个模板的范围内，但是在我们上面的示例中，`$key` 和 `$val` 作用域只在 

```
{{ range... }}{{ end }}
```

 区域内。

> 但是，有一个始终是全局变量的 `$` 始终指向顶层根上下文，当我们在 `range` 循环内需要知道 chart 包的 release 名称的时候，该功能就非常有用了，比如下面的模板文件：

```
{{- range .Values.tlsSecrets }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .name }}
  labels:
    # helm 模板经常使用 `.`，但是这里是无效的，用 `$` 是可以生效的。
    app.kubernetes.io/name: {{ template "fullname" $ }}
    # 这里不能引用 `.Chart.Name`，但是可用使用 `$.Chart.Name`
    helm.sh/chart: "{{ $.Chart.Name }}-{{ $.Chart.Version }}"
    app.kubernetes.io/instance: "{{ $.Release.Name }}"
    # 值来自于 Chart.yaml 文件中的 appVersion
    app.kubernetes.io/version: "{{ $.Chart.AppVersion }}"
    app.kubernetes.io/managed-by: "{{ $.Release.Service }}"
type: kubernetes.io/tls
data:
  tls.crt: {{ .certificate }}
  tls.key: {{ .key }}
---
{{- end }}
```

> 到现在为止，我们只研究了在一个文件中声明的一个模板，但是，Helm 模板语言的强大功能之一是它能够声明多个模板并将其一起使用。我们将在下面的章节中来讨论这一点。