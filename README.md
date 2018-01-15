# 《使用webpack-dev-server实现热更新HMR》

`热更新`

这个也是在github上看到一小哥的课程，觉得很有用，了解下热更新，术语可能讲不来，但是通俗点的意思是
可以实现局部刷新，当我们修改html，css，js脚本时，不用再f5重新刷新页面看效果，

热更新会把我们修改的地方进行局部更新，不用重新刷页面，
这样可以大大的提高我们的工作效率，
而且在有前后台交互数据的情况下，
不用再再去因为改动一个小地方再load一次服务器跟浏览器。
在学习热更新的过程中，明白了开发环境跟正式环境是不同的，
好了，记录下这次学习的过程。


### 安装webpack-dev-server

    npm install webpack-dev-server --save-dev


### 编写配置文件 webpack.config.js
先认真点哈，认真的看下配置文件，里面有一些require 相关的插件，config函数里有多个小函数，
entry，output等等的配置

// 定义webpack-dev-server


```Javascript
//这里引入webpack是为了使用webpack的热更新功能以及其他自带插件，见 module.exports.plugins
var webpack = require('webpack'),
    HtmlWebpackPlugin = require('html-webpack-plugin'),
    ExtractTextPlugin = require("extract-text-webpack-plugin"),
    ImageminPlugin = require('imagemin-webpack-plugin').default;

var path = require('path');
var ROOT_PATH = path.resolve(__dirname);
var BUILD_PATH = path.resolve(ROOT_PATH, './dist');

var config = {
//定义了一些文件夹的路径
    entry: process.env.NODE_ENV === 'production' ? './webpack.entry.js' : [
        // 给webpack-dev-server启动一个本地服务
        'webpack-dev-server/client?http://localhost:8081',
        // 给上面启动的本地服务开启自动刷新功能，'only-dev-server'的'only-'意思是只有当模块允许被热更新之后才有热加载，否则就是整页刷新
        'webpack/hot/only-dev-server',
        './webpack.entry.js'
    ],
    output: {
        filename: 'js/webpack.bundle.js',
        path: BUILD_PATH,
        publicPath: ''
    },
    context: __dirname,
    module: {
        rules: [{
            test: /\.css$/,
            use: process.env.NODE_ENV == 'production' ? ExtractTextPlugin.extract({
                fallback: "style-loader",
                use: "css-loader"
            }) :
                [
                    'style-loader',
                    'css-loader?sourceMap' // 这里需要配置sourceMap参数
                ]
        }, {
            test: /\.(gif|jpg|png|woff|svg|eot|ttf)\??.*$/,
            use: [
                'url-loader?limit=8192&name=./img/[name].[ext]'
            ]
        }, {
            test: /\.html$/,
            use: [
                'html-loader'
            ]
        }]
    },
    plugins: [
        new webpack.EnvironmentPlugin(['NODE_ENV']),
        new HtmlWebpackPlugin({
            template: './index.html',
            filename: 'index.html'
        }),
        new webpack.HotModuleReplacementPlugin(),
        new webpack.NamedModulesPlugin()
    ],
    // 定义webpack-dev-server
    devServer: {
        contentBase: path.resolve(__dirname, 'src'), // 静态文件目录位置，只有当你需要在webpack-dev-server本地服务器查看或引用静态文件时用到。类型：boolean | string | array, 建议使用绝对路径
        hot: true, // 模块热更新。依赖于HotModuleReplacementPlugin
        noInfo: false, // 在命令行窗口显示打包信息
    }
};

if (process.env.NODE_ENV === 'production') {
    config.plugins.push(
        //压缩图片
        new ImageminPlugin({
            pngquant: {
                quality: '70'
            }
        }),
        // 单独打包css
        new ExtractTextPlugin({filename: '[name].css', disable: false, allChunks: true}),
        // js、css都会压缩
        new webpack.optimize.UglifyJsPlugin({
            mangle: {
                except: ['$super', '$', 'exports', 'require', 'module', '_', '*.swf']
            },
            compress: {
                warnings: false
            },
            output: {
                comments: false
            }
        }))
}

module.exports = config;


```

#### 补充，配置文件说明

  有一个问题是，热更新的时候，webpack是不能支持`ExtractTextPlugin`这个插件的，
  但是这个插件是我们在到正式环境，打包的时候才需要用，（css单独打包出来）,不加的话 index.html会把js css 全部打包到bundle.js里，
  这样是不好维护的，继续看，


在plugins插件里我们添加`new webpack.EnvironmentPlugin(['NODE_ENV']),` 这个用来判断当前环境

```Javascript
      plugins: [
            new webpack.EnvironmentPlugin(['NODE_ENV']),
            ..
            ..
            ],

```



> 然后在entry里我们可以看到有一段这样的代码

```Javascript
entry: process.env.NODE_ENV === 'production' ? './webpack.entry.js' : [
       // 给webpack-dev-server启动一个本地服务
        'webpack-dev-server/client?http://localhost:8081',
       // 给上面启动的本地服务开启自动刷新功能，
       // 'only-dev-server'的'only-'意思是只有当模块允许被热更新之后才有热加载，否则就是整页刷新
        'webpack/hot/only-dev-server',
        './webpack.entry.js'
    ],
```

我们在entry 入口里，判断当下环境，
如果是开发环境就会启动webpack-dev-server，是正式环境就直接打包，然后在配置文件里配置devServer，

```Javascript
// 定义webpack-dev-server
    devServer: {
        contentBase: path.resolve(__dirname, 'src'), // 静态文件目录位置，只有当你需要在webpack-dev-server本地服务器查看或引用静态文件时用到。类型：boolean | string | array, 建议使用绝对路径
        hot: true, // 模块热更新。依赖于HotModuleReplacementPlugin
        noInfo: false, // 在命令行窗口显示打包信息
    }
```

在最后面如果是正式环境我们会把图片压缩，加了`ExtractTextPlugin` ，`ImageminPlugin` ，`UglifyJsPlugin` 这三个插件
功能用法在下面注释里有写，

```Javascript
if (process.env.NODE_ENV === 'production') {
    config.plugins.push(
        //压缩图片
        new ImageminPlugin({
            pngquant: {
                quality: '70'
            }
        }),
        // 单独打包css
        new ExtractTextPlugin({filename: '[name].css', disable: false, allChunks: true}),
        // js、css都会压缩
        new webpack.optimize.UglifyJsPlugin({
            mangle: {
                except: ['$super', '$', 'exports', 'require', 'module', '_', '*.swf']
            },
            compress: {
                warnings: false
            },
            output: {
                comments: false
            }
        }))
}

```

### webpack会监听被require或import的文件，copy以下代码到webpack.entry.js

如果是开发环境就把index也加入，如果开发环境，我们也需要判断html页面的修改，实现热更新，
我们的配置文件里加了`HtmlWebpackPlugin`，html打包，判断环境是为了打包的时候 不会重复资源，

```Javascript 
if (process.env.NODE_ENV === 'development') {
    require('./index.html');
}

require('./css/loginCompont.css');
require('./css/main.css');
require('./spr/sprite.css');
require("./js/main");


```

### package.json 

在这个里面我们看到有

```Javascript
 "scripts": {
    "start": "cross-env NODE_ENV=development node_modules/.bin/webpack-dev-server",
    "build": "cross-env NODE_ENV=production  node_modules/.bin/webpack"
  },

```

`cross-env  NODE_ENV=development `   
`cross-env  NODE_ENV=productio ` 

由于windows不支持NODE_ENV=development的设置方式。

解决方式是用cross-env
这个迷你的包能够提供一个设置环境变量的scripts，让你能够以unix方式设置环境变量，然后在windows上也能兼容运行。

使用方法：
安装cross-env:   
   
    npm install cross-env --save-dev
    
在NODE_ENV=xxxxxxx前面添加cross-env就可以了。


运行npm start命令，修改保存index.html，页面自动刷新，完美～～～
至此，咱们已经可以随意修改html、css、js、图片等文件并实现页面自动热更新了。 

