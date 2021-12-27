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

