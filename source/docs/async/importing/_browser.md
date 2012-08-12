在浏览器里使用Wind.js异步模块时，只需引入[bin/dev](https://github.com/JeffreyZhao/wind/tree/master/bin/dev)目录下的相关文件即可：

    <script src="wind-core-x.y.z.js"></script>
    <script src="wind-compiler-x.y.z.js"></script>
    <script src="wind-builderbase-x.y.z.js"></script>
    <script src="wind-async-x.y.z.js"></script>

此时将会在浏览器根上创建一个名为`Wind`的对象。而在生产环境里部署时，建议在[预编译]({{root_url}}/docs/aot/)后去除体积最为庞大的compiler模块。更多内容请参考“[模块引入]({{root_url}}/docs/importing/)”相关文档。