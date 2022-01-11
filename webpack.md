# webpack

## webpack简介

> 一种前端资源构建工具，一个静态模块打包器（module bundler）

> 根据入口文件的依赖关系，将资源引进来，形成chunk代码块，根据不同资源进行编译买这个处理过程我们叫打包，打包输出的文件叫bundle;
将所有资源、静态文件（js、less、图片等），也被称为模块，根据模块的依赖关系进行静态分析，通过打包操作生成一个可以直接在浏览器中执行的文件bundle，这个操作的过程使用的工具就是webpack，也就是静态模块打包器的意思由来。

## 运行原理

> 1.读取webpack.config.js中的配置文件，生成compiler实例，并把compiler实例注入plugin中的apply方法中
> 2.读取配置中的entries，递归遍历所有的入口文件
> 3.对入口文件进行编译，开始compilation过程，使用loader对文件内容进行编译，再将编译好的文件内容解析成AST静态语法树
> 4.递归依赖的模块，重复第三步，生成AST语法树，在AST语法树中可以分析到模块之间的依赖关系，对应做出优化
> 5.将所有模块中的require语法替换成_webpack_require_来模拟模块化操作
> 6.最后把所有模块打包到一个自执行函数中

(还需理解理解)

---

## webpack五个核心概念

> **entry**：用来告诉webpack从哪个文件作为入口开始打包，打包前会分析构建内部依赖图
```
    module.exports={
        entry:'./path/to/file.js' 
    }
```

> **output**：告诉webpack将打包后的资源bundle输出到哪里（path\publicPath）,以及叫什么名字（filename）
```
    const path = require("path");
    module.exports = {
        entry: "./path/to/my/entry/file.js",
        output: {
            path: path.resolve(__dirname, "dist"),
            filename: "my-first-webpack.bundle.js"
        }
    };
```

> **loader**：让webpack能够处理那些为js文件，因为webpack自身只能理解js，像css、img等文件无法处理，这时候就需要loader来将这些文件翻译成webpack能看懂的内容，从而使webpack能去处理
```
    const path = require("path");
    module.exports = {
        output: {
            filename: "my-first-webpack.bundle.js"
        },
        module: {
            rules: [
                {
                    // 根据后缀名匹配需要处理的文件
                    test: /\.txt$/,
                    // 使用对应的loader处理文件
                    use: "raw-loader"
                }
            ]
        }
    };
```

> **plugins**：（插件）用于执行范围更广的任务，因为loader只能作为翻译，更多其他的功能无法实现，这时候就需要插件来做功能更强大的事，插件的范围包括，从打包优化和压缩，一直到重新定义环境中的变量等。
```
    const HtmlWebpackPlugin = require("html-webpack-plugin");
    module.exports = {
        module: {
            rules: [{ test: /\.txt$/, use: "raw-loader" }]
        },
        plugins: [new HtmlWebpackPlugin({ template: "./src/index.html" })]
    };
```

> 编写自定义插件
利用webpack提供的钩子函数，编写自定义插件，相当于监听webpack的事件，并作出相应的响应。
```
    class APlugin {
        // apply方法，会在new plugin后被webpack自动执行。
        apply(compiler) {
            // 可以在任意的钩子函数中去触发自定义事件，也可以监听其他事件：compiler.hooks.xxxx
            compiler.hooks.compilation.tap("APlugin", compilation => {
                compilation.hooks.afterOptimizeChunkAssets.tap("APlugin", chunks => {
                    // 这里只是简单的打印了chunks，你如果有更多的想法，都可以在这里实现。
                    console.log("打印chunks：", chunks);
                });
            });
        }
    }
```
如果不使用plugins，webpack会把所有文件打包到一个js文件中，随着项目的迭代和内容的增大，这个文件也会变得很大，从而导致加载时间变得很长，这个时候需要去配置optimization.splitChunks来设置拆分文件的规则，从而解决上述的问题。
> webpack默认的配置
```
    module.exports = {
        optimization: {
            splitChunks: {
                chunks: "async", // 参数可能是：all，async和initial，这里表示拆分异步模块。
                minSize: 30000, // 如果模块的大小大于30kb，才会被拆分
                minChunks: 1,
                maxAsyncRequests: 5, // 按需加载时最大的请求数，意思就是说，如果拆得很小，就会超过这个值，限制拆分的数量。
                maxInitialRequests: 3, // 入口处的最大请求数
                automaticNameDelimiter: "~", // webpack将使用块的名称和名称生成名称（例如vendors~main.js）
                name: true, // 拆分块的名称
                cacheGroups: {
                    // 缓存splitchunks
                    vendors: {
                        test: /[\\/]node_modules[\\/]/,
                        priority: -10
                    },
                    default: {
                        minChunks: 2, // 一个模块至少出现2次引用时，才会被拆分
                        priority: -20,
                        reuseExistingChunk: true
                    }
                }
            }
        }
    };
```

> **mode**：（模式）分为development和production，其中development（开发模式）能让代码在本地调试的运行环境，production（生产模式）为让代码优化上线运行的环境。

> 在webpack构建的过程中，主要花费时间的部分是递归遍历各个entry，然后寻找依赖并且逐个依次编译的过程，每次递归都要经历String->AST->String的流程（emmmmmm，不是很懂这个），经过loader还需要处理一些字符串或者执行一些js脚本，同时node.js还是单线程的。因此需要使用happypack，happypack是使用了node processes执行多线程构建，可以让多个loader并行执行，从而加快webpack的构建。
```
    // @file: webpack.config.js
    var HappyPack = require("happypack");
    var happyThreadPool = HappyPack.ThreadPool({ size: 5 });
    module.exports = {
        // ...
        plugins: [
            new HappyPack({
                id: "jsx",
                threadPool: happyThreadPool,
                loaders: ["babel-loader"]
            }),
            new HappyPack({
                id: "styles",
                threadPool: happyThreadPool,
                loaders: ["style-loader", "css-loader", "less-loader"]
            })
        ]
    };
    exports.module.rules = [
        {
            test: /\.js$/,
            use: "happypack/loader?id=jsx"
        },
        {
            test: /\.less$/,
            use: "happypack/loader?id=styles"
        }
    ];
```

---
## webpack性能优化

### 缩小文件搜索范围
> 1.通过exclude、include 缩小搜索范围
例如：
```
module.exports = {
    module:{
        rules:[
            {
                test:/\.js$/,
                loader:&apos;babel-loader&apos;,
                // 只在src文件夹中查找
                include:[resolve(&apos;src&apos;)],
                // 排除的路径
                exclude:/node_modules/
            }
        ]
    }
}
```
> 2. 合理利用resolve 字段配置
    2.1 配置`resolve.modules:[path.resolve(__dirname,&apos;node_modules&apos;)]`避免层层查找
        其中 resolve.modules会告诉webpack去哪些目录寻找第三方模块，如果不配置 `path.resolve(__dirname,&apos;node_modules&apos;)`，则会依次查找.`/node_module、../node_modules`，一层一层网上找，这显然效率不高。
    2.2对庞大的第三方模块设置 resolve.alias，使webpack直接使用库的min文件，避免库内解析
       可以通过别名的方式来映射一个路径，能让Webpack更快找到路径。
       例如：
       ```
       resolve.alias:{
            &apos;react&apos;:patch.resolve(__dirname, &apos;./node_modules/react/dist/react.min.js&apos;)
        }

       ```
    2.3 resolve.extensions ，减少文件查找
        resolve.extensions 用来表明文件后缀列表，默认查找顺序是：[&apos;.js&apos;,&apos;.json&apos;]，如果导入文件没有添加后缀就会按照这个顺序查找文件。应该尽可能减少后缀列表长度，然后将出现频率高的后缀排在后面。
        
### 缓存之前构建过的js
> 将Babel编译过的文件缓存起来，下次只需要编译更改过的代码文件即可，这样可以大幅度加快打包时间。
> loader:&apos;babel-loader?cacheDirectory=true&apos;

### 提前构建第三方库
> 处理第三方库的方法有很多种，其中，Externals不够聪明，一些情况下会引发重复打包的问题；而 CommonsChunkPlugin 每次构建时都会重新构建一次 vendor；处于效率考虑还是考虑使用DllPlugin。
DLL全称Dynamic-link library，（动态链接库）。到底怎么个动态法。原理是将网页依赖的基础模块抽离出来打包到dll文件中，当需要导入的模块存在于某个dll中时，这个模块不再被打包，而是去dll中获取，而且通常都是第三方库。那么为什么能提升构建速度，原因在于这些第三方模块如果不升级，那么只需要被构建一次。

### 并行构建而不是同步构建
> 受限于 Node 是单线程运行的，所以 Webpack 在打包的过程中也是单线程的，特别是在执行 Loader 的时候，长时间编译的任务很多，这样就会导致等待的情况。HappyPack和ThreadLoader作用是一样的，都是同时执行多个进程，从而加快构建速度。而Thread-Loader是webpack4提出的。

### html文件处理
> 默认不能使用HMR功能，而且对于html文件一般也不做热更新。对于单页面来讲，因为一旦html更新了，那么整个页面都刷新了，称不上真正意义的热更新。

### css文件处理
> 默认情况下css可以使用HMR功能，因为style-loader内部实现了。

### js文件处理
> 默认不能使用HMR功能，需要修改js代码，添加支持HMR功能代码
> 而且，HMR功能对js的处理，只能处理非入口js文件。


## 压缩打包体积

### 删除冗余代码
> 使用TreeShaking删除无用代码，这里的无用指的是当发现引入模块的某些内容在其他地方并没有使用时，就被当作无用节点，从而被删掉。看起来时高级的技术，但是在有些版本也有被误杀的可能性。
> 使用前提:依赖ES6的import、export模块化语法、webpack.config.js 的 mode:&apos;production&apos;


### 代码分割实现按需加载
（待续）








