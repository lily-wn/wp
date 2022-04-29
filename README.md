# wp提取公共代码
压缩代码
Tree Shaking
Code Splitting代码分割

提取公共代码
提取公共代码一般用于多入口的情况，为防止重复打包而进行的优化操作，好处是可以减少文件体积，加快打包和启动速度


提取js公共代码：
项目中我们常会遇到有多个入口文件的情况（假设为a.js与b.js），如果每个入口文件都引用了相同的模块（假设有自定义模块tools.js与第三方模块lodash），文件代码如下：
    // a.js
    import _ from 'lodash'
    import tools from './utils/tools.js'
复制代码
    // b.js
    import _ from 'lodash'
    import tools from './utils/tools.js'
复制代码
webpack配置如下：
    module.exports = {
        // ...其它配置
        entry:{
            a:'./src/a.js',
            b:'./src/b.js'
        },
        output:{
            path: path.join(__dirname, "dist"),
            filename:'[name].bundle.js'
        }
    }
复制代码
打包后的效果如下：

在打包时默认就会把tools.js与lodash打包两次，这样既增加文件体积（两个文件都是536k），既影响性能，还降低了我们的代码质量，这时我们可以使用提取公共代码的方式来进行优化
优化后的webpack配置如下：

使用optimization.splitChunks.cacheGroups实现提取公共代码（更多配置说明请查看: webpack.docschina.org/plugins/spl…

    module.exports = {
        entry:{
            a:'./src/a.js',
            b:'./src/b.js'
        },
        output:{
            path: path.join(__dirname, "dist"),
            filename:'[name].bundle.js'
        },
        optimization:{
            splitChunks:{
                cacheGroups:{
                    // 注意: 
                    // 这里的key命名自定义
                    // priority：值越大优先级越高
                    // chunks指定哪些模块需要打包，可选值有
                    //  * initial: 初始块
                    //  * async: 按需加载块(默认)
                    //  * all: 全部块
                    
                    // common: 打包业务中公共代码（上面的tools.js）
                    common: {
                      name: "common", // 指定包名，不指定时使用上层key作为包名
                      chunks: "all", 
                      minSize: 10,
                      priority: 0
                    },
                    // vendor: 打包node_modules中的文件（上面的 lodash）
                    vendor: {
                      name: "vendor",
                      test: /node_modules/,
                      chunks: "all",
                      priority: 10
                    }
                }
            }
        }
    }
复制代码

打包后的结果是，lodash被打包到vendor.bundle.js, tools.js被打包到common.bundle.js，从而实现了公共代码的提取。


利用externals提取第三方库
准确来说应是该通过externals配置告诉webpack哪些第三方库不需要打包到bundle中，实际开发中有一些第三方库（如jQuery）,如果我们在项目中使用以下代码引入，打包时jQuery就会被打包到最终的bundle中
    import $ from 'jquery'
复制代码
但有时我们更希望使用CDN来引入jQuery，webpack给我们提供的externals可以轻松实现，步骤如下：

在html文件中引入cdn库链接

    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title><%= htmlWebpackPlugin.options.title %></title>
    </head>
    <body>
        <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
    </body>
    </html>
复制代码

webpack配置

    module.exports = {
        // ...其它选项
        externals: {
            jquery: "jQuery",
        }
    }
复制代码

使用jQuery

    import $ from 'jquery'
    $('h1').css('color','#58bc58')
复制代码
最终打包效果可以看出，webpack只是做了个简单的导出，导出全局作用域中的jQuery变量，而这个变量在script中引入jQuery时就已存在

举一反三，我们自定义的一些工具库，如果不需要打包到最终的bundle中，我们也可以采用这样方式。


提取css公共代码
前面讲到的css-loader与style-loader只是把css样式写入到html页面的style标签内，如果是SPA单页面应用，这没什么问题，但如果是多页面应用，则会在每个页面中都写入这样css样式，那我们能不能把这些相同的部分提取出来，然后使用link去引入页面呢？答案是肯定的，只需要使用mini-css-extract-plugin插件就可以实现


安装插件
    npm install mini-css-extract-plugin -D
复制代码


配置webpack
在配置时，除了配置plugins选项，还需要在loader中进行配置，因为是提取css到单独文件，所以删除原来的style-loader，改成MiniCssExtractPlugin.loader
    const MiniCssExtractPlugin = require("mini-css-extract-plugin");

    module.exports = {
        plugins: [new MiniCssExtractPlugin()],
        module: {
            rules: [
            {
                test: /\.css$/i,
                use: [MiniCssExtractPlugin.loader, "css-loader"],
            },
            ],
        },
    };
复制代码




压缩代码
实现删除多余的代码、注释、简化代码的写法等⽅式。


压缩JS：
Webpack默认在生产环境下（mode:'production'）自动进行代码压缩，内部用的是terser-webpack-plugin插件，具体使用请查看官网


压缩css:
要压缩css，前提是先提取css到独立文件中（前面讲过），然后通过css-minimizer-webpack-plugin插件实现压缩CSS，该插件基于cssnano实现css代码的压缩优化


安装
    npm install css-minimizer-webpack-plugin -D
复制代码


配置webpack
注意该插件不是写在plugins中，而是写在优化配置optimization.minimizer中，且mode必须为production
    const CssMinimizerPlugin = require("css-minimizer-webpack-plugin");
    module.exports = {
        mode:'production',
        optimization:{
            minimizer:[
                new CssMinimizerPlugin(),
            ]
        }
    }
复制代码






Tree Shaking
Tree Shaking 也叫摇树优化，是一种通过移除冗余代码，来优化打包体积的手段，它并不是webpack中的某个配置选项，而是一组功能搭配使用后的效果，基于ESModules模块化（即只有ESModules的模块化代码才能使Tree Shaking生效），在production生产环境下默认开启

举个栗子，假设我有一个element.js文件，代码如下
    // element.js
    export const Button =()=>{
        return document.createElement('button')
        // 不可能执行的代码
        console.log('end')
    }

    //未引用的代码
    export const Link=()=>{
        return document.createElement('a')
    }
复制代码
然后引入这个模块并使用模块中的方法
    import {Button} from './element.js'
    const btn = Button();
    document.body.appendChild(btn);
复制代码
在mode:'production'模式下打包的效果如下：

显然，那些没有使用的代码都不会被打包，达到了优化的效果
那webpack是如何实现的呢，接下来我们不开启production模式的情况下一步步实现


usedExports: 只导出被使用的成员
    module.exports:{
        // ... 省略其它选项
        mode:'none',
        optimization:{
            usedExports:true
        }
    }
复制代码

从上图可以看出，只有Button被导出了，如果你跟我一样用的是VSCode，可以看到Link是暗色的，说明没有被用到，如果要Link代码删除，可以使用下面的minimize


minimize: 压缩后删除不被使用的代码
    module.exports:{
        // ... 省略其它选项
        mode:'none',
        optimization:{
            usedExports:true,
            minimize:true
        }
    }
复制代码
打包后，在最终的代码中就已经没有Link代码了，如下图：



concatenateModules: 尽可能合并每一个模块到一个函数中
正常的打包效果是每个模块代码都放在一个单独的函数中，如果引入的模块很多就会出现很多函数，这样会影响执行效率和打包后的文件体积大小，可以通concatenateModules:true把多个模块的代码合并到一个函数中，大家可以自行测试效果


sideEffects: 指定副作用代码
Tree Shaking会自动删除模块中一些没有被引用的代码，但这个行为在某些场景下可能会出现问题，比如extend.global.js模块代码如下
    // 实现首字母大写
    String.prototype.capitalize = function(){
        return this.split(/\s+/).map(function(item){
            return item[0].toUpperCase()+item.slice(1)
        }).join(' ')
    }
复制代码
使用代码如下，由于该模块没有任何导出，只需要引入就可以使用capitalize()方法了，这种代码我们称为副作用代码
    import './extend.global.js'
    'hello boy'.capitalize(); // Hello Boy
复制代码
Webpack4默认把所有的代码看作副作用代码，所以会把所有的代码都打包到最终结果中，当然这样的后果是会把一些多余的代码也打包进来导致文件过大。而Webpack5默认开启Tree Shaking，前面已经说到了，Tree Shaking功能会自动删除无引用的代码，上面的代码没有任何导出和使用，所以Webpack5不会把extend.global.js中的代码打包进来，结果会导致找不到capitalize()方法而报错，sideEffect就是用来解决此类问题的，用法分两步，代码如下：

optimization.sideEffects设置为true：告知 webpack 去辨识 package.json 中的副作用标记或规则（默认值为true，所以这一步也可以不设置）
package.json添加sideEffects属性，可以为以下值：

true: 告诉webpack所有的模块都是副作用模块

如设置为true，上面的extend.global.js会被打包到最终结果


false: 告诉webpack所有的模块都没有副作用

如设置为false，上面的extend.global.js不会被打包到最终结果，代码就会报错


Array: 手动指定副作用文件

使用true或false会走向两个极端，不一定适合真实的开发场景，可以设置数组来指定副作用文件，代码如下：

    {
        sideEffects:["*.css","*.global.js"]
    }
复制代码
配置后，webpack打包时遇到css文件或以global.js结尾的文件时会自动打包到最终结果





Code Splitting代码分割
把项目中的资源模块按照我们设定的规则打包到不同的文件中，代码分割后可以降低应用启动成本，提高响应速度
1、配置多入口，输出多个打包文件
    module.exports = {
        entry:{
            home:'./src/index.js',
            login:'./src/home.js'
        },
        output:{
            // name: 入口名字
            filename:'[name].bundle.js'
        }
    }
复制代码
两个入口文件代码如下：
    // home.js
    import {formatDate} from './utils'
    import './css/home.css'
    console.log('index',formatDate())
复制代码
     // login.js
    import {formatDate} from './utils'
    import _ from 'lodash'
    console.log('login',formatDate())

    const user = {username:'laoxie',password:123456}
    const copyUser = _.cloneDeep(user)
    console.log(user == copyUser)
复制代码
打包后看到，两个入口文件已经在html文件中自动引入 ：

输出的文件大小也很正常，login.js由于引入了lodash，所以文件比较大

但如果home.js和login.js都引入lodash会是什么结果呢？如下图，我们看到最终打包出来的两个js文件都比较大，很显示webpack重复打包了lodash（即home.bundle.js与login.bundle.js都把lodash打包进去了）

解决重复打包问题
接下来，我们使用dependOn和splitChunks两种方法解决重复打包问题


方法一：dependOn: 通过修改entry入口配置实现，利用dependOn的提取公共的lodash
    module.exports = {
        entry:{
            home:{
                import:'./src/index.js',
                dependOn:'common'
            },
            login:{
                import:'./src/login.js',
                dependOn:'common'
            },
            common:'lodash'
        }
   }
复制代码
打包后的结果如下，成功提取了common.bundle.js（lodash代码），home.bundle.js与login.bundle.js就变得很小了，html文件也成功引入了这3个js文件




方法二：Webpack内置功能splitChunks（推荐）
    module.exports = {
        // ...其它配置
        entry:{
            home:'./src/index.js',
            login:'./src/login.js'
        },
        optimization:{
            // 拆分代码
            splitChunks:{
                chunks:'all'
            }
        }
    }
复制代码
只需要配置optimization.splitChunks.chunks选项，表示哪些代码需要优化，值可以为以下三种：

initial: 初始块
async: 按需加载块(默认)
all: 全部块

我们选择全部优化，打包后同样会输出三个文件，并正常在html文件中引入，如下图





注意：不管使用哪种方式，如果是多页面应用，为防止html文件重复引入，都需要在html-webpack-plugin插件中配置chunks选项

   plugins:[
        new HtmlWebpackPlugin({
            chunks:['common','home']
        }),
        new HtmlWebpackPlugin({
            filename:'login.html',
            chunks:['common','login']
        }),
    ]
复制代码
2、ESModules动态导入（Dynamic Imports）
使用ECMAScript2019（ES10）推出的import()动态引入模块，返回Promise对象，用法如下
    import('lodash').then(({default:_})=>{
        // 引入lodash模块的default属性
    })
复制代码
只要在代码中使用通过以上方式使用lodash，webpack打包时就会对lodash进行独立打包，如下图：


注意：用import()动态引入的模块跟通过import ... from ...静态引入的模块不同处在于模块不会直接写在html文件中，而是在打开页面时才引入，配合用户的行为（如点击等操作）就已经能实现赖加载功能了


而且大家也看到了，生成的公共模块文件名很长（随着模块的增多会更长），我们可以使用webpack的魔法注释 来解决这个问题
     import(/*webpackChunkName:'common'*/'lodash').then(({default:_})=>{
        
    })
复制代码


PS: 利用Webpack的魔法注释还能实现预加载功能，只需要添加webpackPrefetch:true即可，

赖加载：进入页面时不加载，使用到某个模块的时候才加载的方式
预加载：进入页面时先加载，使用时直接调用的方式

Webpack的预加载是在页面其它内容全部加载完成后才加载需要的模块，所以不会影响页面的加载速度



