
## 内置对象

> 前面我们介绍了 Helm Chart 的一些基本概念和使用，接下来我们重点介绍下 Chart 模板的编写。模板会渲染成 Kubernetes 的资源清单文件，下面我们将来学习下模板的结构，如何使用它们，如何编写 Go 模板以及如何调试。

> 对象从模板引擎传递到模板中，在代码中可以传递对象，也有一种方法可以在模板宏创建新的对象，比如 `tuple` 函数。对象可以很简单，也可以包含其他对象或函数，例如，Release 对象就包含几个对象（比如 Release.Name），Files 对象就包含几个函数。

> 前面提到过我们可以在模板中使用 `{{ .Release.Name }}` 获取 release 的名称，Release 是我们可以在模板中访问的几个顶级对象之一：

*   > `Release`：该对象描述了 release 本身的相关信息，它内部有几个对象：

    *   `Release.Name`：release 名称
    *   `Release.Namespace`：release 安装到的命名空间
    *   `Release.IsUpgrade`：如果当前操作是升级或回滚，则该值为 true
    *   `Release.IsInstall`：如果当前操作是安装，则将其设置为 true
    *   `Release.Revision`：release 的 revision 版本号，在安装的时候，值为1，每次升级或回滚都会增加
    *   `Reelase.Service`：渲染当前模板的服务，在 Helm 上，实际上该值始终为 Helm
*   > `Values`：从 `values.yaml` 文件和用户提供的 values 文件传递到模板的 Values 值，默认情况下，Values 是空的。

*   `Chart`：获取 `Chart.yaml` 文件的内容，该文件中的任何数据都可以访问，例如 

    ```
    {{ .Chart.Name }}-{{ .Chart.Version}}
    ```

     可以渲染成 `mychart-0.1.0`，该对象下面可用的字段前面我们已经提到过了。
*   > `Files`：可以访问 chart 中的所有非特殊文件，虽然无法使用它来访问模板文件，但是可以来访问 chart 中的其他文件。

    *   `Files.Get`：用于根据名称获取文件（比如 

        ```
        .Files.Get config.ini
        ```

        ）
    *   `Files.GetBytes`：用于以 bytes 数组而不是字符串的形式来获取文件内容的函数，这对于类似于图片之类的东西很有用
    *   `Files.Glob`：用于返回名称于给定的 shell glob 模式匹配的文件列表
    *   `Files.Lines`：可以逐行读取文件的函数，对于遍历文件中的每行内容很有用
    *   `Files.AsSecrets`：将文件内容以 Base64 编码的字符串返回的函数
    *   `Files.AsConfig`：将文件正文作为 YAML 字典返回的函数
*   > `Capabilities`：提供了获取有关 Kubernetes 集群支持功能的信息的对象

    *   ```
        Capabilities.APIVersions
        ```

        ：支持的版本集合
    *   ```
        Capabilities.APIVersions.Has $version
        ```

        ：判断一个版本（比如 `batch/v1`）或资源（比如 `apps/v1/Deployment`）是否可用
    *   ```
        Capabilities.Kube.Version
        ```

        ：Kubernetes 的版本
    *   `Capabilities.Kube`：是 Kubernetes 版本的缩写
    *   ```
        Capabilities.Kube.Major
        ```

        ：Kubernetes 主版本
    *   ```
        Capabilities.Kube.Minor
        ```

        ：Kubernetes 的次版本
*   > `Template`：包含当前正在执行的模板的相关信息

    *   `Name`：当前模板的命名空间文件路径（比如 

        ```
        mychart/templates/mytemplate.yaml
        ```

        ）
    *   `BaePath`：当前 chart 的模板目录的命名空间路径（比如 `mychart/templates`）

> 需要注意的是内置的对象始终是以大写字母开头的，这也是符合 Go 的命名约定的，创建自己的名称的时候，可以自由使用适合你团队的约定，一些团队，比如 Kubernetes Charts 团队，选择仅使用首字母小写，以区分本地名称和内置名称，这里我们也会遵循该约定。