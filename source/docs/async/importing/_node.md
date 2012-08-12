在Node.js里使用Wind.js异步模块时，只需使用npm安装wind包，其中包含了Wind.js开发中所需的所有模块。：

    npm install wind
    
并在需要时使用`require`引入即可：

    var Wind = require("wind");

在Wind.js使用过程中，必须保证在上下文环境中有一个名为`Wind`的根对象。