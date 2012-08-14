在浏览器里使用Wind.js异步模块开发应用程序时，只需引入[bin/dev](https://github.com/JeffreyZhao/wind/tree/master/bin/dev)目录下的wind-all脚本即可，其中包含了Wind.js的完整代码：

    <script src="wind-all-x.y.z.js"></script>

不过严格说来，您只需引入以下几个文件也可以得到相同的效果：

    <script src="wind-core-x.y.z.js"></script>
    <script src="wind-compiler-x.y.z.js"></script>
    <script src="wind-builderbase-x.y.z.js"></script>
    <script src="wind-async-x.y.z.js"></script>

此时将会在浏览器根上创建一个名为`Wind`的对象。值得注意的是，在生产环境里部署时，建议在[预编译]({{root_url}}/docs/aot/)后去除体积最为庞大的compiler模块。更多内容请参考“[模块引入]({{root_url}}/docs/importing/)”相关文档。