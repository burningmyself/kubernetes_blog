
## 模板调试

> 调试模板可能比较麻烦，因为渲染的模板会发送到 Kubernetes API server，而 API server 可能会因为格式以外的一些原因而拒绝 YAML 文件。

> 下面这些命令可以帮助你调试一些问题：

*   `helm lint` 是验证 chart 是否遵循最佳实践的首选工具
*   ```
    helm install --dry-run --debug
    ```

     或者 

    ```
    helm template --debug
    ```

    ：前面我们已经使用了这个技巧，这个是让服务器渲染模板，然后返回生成的资源清单文件的好方法，而且不会真正的去安装这些资源
*   `helm get manifest`：这是查看服务器上安装了哪些模板的好方法

> 当你的 YAML 文件无法解析的时候，但你想要查看生成的内容的时候，检索 YAML 的一种简单方法是注释掉模板中的问题部分，然后重新运行 

```
helm install --dry-run --debug
```

：

```
apiVersion: v2
# some: problem section
# {{ .Values.foo | quote }}
```

> 上面的内容将呈现并返回完整的注释：

```
apiVersion: v2
# some: problem section
#  "bar"
```

> 这提供了一种查看生成的内容的快速方法。