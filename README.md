# Webpack学习总结 #
webpack, 更优秀的前端模块依赖管理工具。

## What is webpack ##
### 网上介绍 ###
webpack是近期最火的一款模块加载器兼打包工具，它能把各种资源，例如JS（含JSX）、coffee、样式（含less/sass）、图片等都作为模块来使用和处理。
![what is webpack](http://webpack.github.io/assets/what-is-webpack.png)

### require ###
模块依赖，一招搞定

	require("./lib.js");
	require("./style.css");
	require("./style.less");
	require("./template.jade");
	require("./image.png");

在 Webpack 当中, 所有的资源都被当作是模块。

### 加载器 ###
对应各种不同文件类型的资源，Webpack有对应的模块loader

	module: {
		//加载器配置
		loaders: [
			//.css 文件使用 style-loader 和 css-loader 来处理
			{ test: /\.css$/, loader: 'style-loader!css-loader' },
			//.js 文件使用 jsx-loader 来编译处理
			{ test: /\.js$/, loader: 'jsx-loader?harmony' },
			//.scss 文件使用 style-loader、css-loader 和 sass-loader 来编译处理
			{ test: /\.scss$/, loader: 'style!css!sass?sourceMap'},
			//图片文件使用 url-loader 来处理，小于8kb的直接转为base64
			{ test: /\.(png|jpg)$/, loader: 'url-loader?limit=8192'}
		]
	}

## webpack的优势 ##
1. webpack 是以 commonJS 的形式来书写脚本滴，但对 AMD/CMD 的支持也很全面，方便旧项目进行代码迁移
2. 所有静态资源都可以被当成模块引用，而不仅仅是JS了
3. 开发便捷，能替代部分 grunt/gulp 的工作，比如打包、压缩混淆、图片转base64等
4. 扩展性强，插件机制完善，特别是支持 React 热插拔（见 react-hot-loader ）的功能让人眼前一亮

以 AMD/CMD 模式来说，鉴于模块是异步加载的，所以我们常规需要使用 define 函数来帮我们搞回调：

	define(['package/lib'], function(lib){
		function foo(){
			lib.log('hello world!');
		} 
		return {
			foo: foo
		};
	});

另外为了可以兼容 commonJS 的写法，我们也可以将 define 这么写：

	define(function (require, exports, module){
		var module1 = require("module1");
		var module2 = require("module2");    
	 
		module1.sayHello();
		module2.sayHi();
	 
		exports.helloWorld = function (){
			module1.sayHello();
			module2.sayHi();
		};
	});

然而对 webpack 来说，我们可以直接在上面书写 commonJS 形式的语法，无须任何 define （毕竟最终模块都打包在一起，webpack 也会最终自动加上自己的加载器）：

	var module1 = require("module1");
	var module2 = require("module2");    
 
	module1.sayHello();
	module2.sayHi();
 
	exports.helloWorld = function (){
		module1.sayHello();
		module2.sayHi();
	};

不过即使你保留了之前 define 的写法也是可以滴，毕竟 webpack 的兼容性相当出色，方便你旧项目的模块直接迁移过来。

## 安装使用 ##
### 安装webpack ##
首先确保机子上已安装node.js，然后通过npm安装webpack

```$
npm install webpack -g
```

### 启动命令 ###

切换到有 webpack.config.js 的目录然后运行

```$
webpack     // 执行一次开发的编译
webpack -p  // 针对发布环境编译(压缩代码)
webpack -w  // 进行开发过程持续的增量编译(飞快地!)
webpack -d  // 生成map映射文件，告知哪些模块被最终打包到哪里了
webpack --config XXX.js   //使用另一份配置文件（比如webpack.config2.js）来打包
```

### 配置文件(webpack.config.js) ###
每个项目下都必须配置有一个 webpack.config.js
- plugins 插件项
- entry 页面入口文件配置
- output 对应输出项配置（即入口文件最终要生成什么名字的文件、存放到哪里）
- module.loaders 最关键的一块，配置每一种文件需要使用什么加载器来处理（多个loader之间用"!"连接）

#### 插件的安装 ###
所有的加载器都需要通过 npm 来加载，并建议查阅它们对应的 readme 来看看如何使用

	npm install url-loader --save-dev

如果目录没有package.json，则需要先init一下，再运行`npm install`命令

	npm init
	npm install url-loader --save-dev

### 通用配置文件例子 ###

	// webpack.config.js
	var webpack = require('webpack');
	var commonsPlugin = new webpack.optimize.CommonsChunkPlugin(/* chunkName= */'common', /* filename= */'common.js'); // 分析以下模块的共用代码, 单独打一个包到common.js
	var ExtractTextPlugin = require("extract-text-webpack-plugin"); // 单独打包CSS
	var HtmlWebpackPlugin = require('html-webpack-plugin'); // Html文件处理
	
	module.exports = {
		entry: {
			Detail: './modules/app/detail.js',
			Home: './modules/app/home.js'
		},
		output: {
			path: './build', // This is where images & js will go
			//publicPath: 'http://m.pp.cn/ppaweb/test/build/', // This is used to generate URLs to e.g. images
			publicPath: '/ppaweb/example/build/', // This is used to generate URLs to e.g. images
			filename: '[name].js',
			chunkFilename: "[id].chunk.js?[hash:8]"
		},
		plugins: [
			commonsPlugin,
			new ExtractTextPlugin('[name].css', {allChunks: true}), // 单独打包CSS
	
			// 全局变量
			new webpack.DefinePlugin({
				//__DEV__: JSON.stringify(JSON.parse(process.env.BUILD_DEV||'false')) //通过环境变量设置
				__DEV__: 'false' // 开发调试时把它改为true
			}),
	
			/**
			* HTML文件编译，自动引用JS/CSS
			* 
			* filename - 输出文件名，相对路径output.path
			* template - HTML模板，相对配置文件目录
			* chunks - 只包含指定的文件（打包后输出的JS/CSS）,不指定的话，它会包含生成的所有js和css文件
			* excludeChunks - 排除指定的文件（打包后输出的JS/CSS），比如：excludeChunks: ['dev-helper']
			* hash
			*/
			new HtmlWebpackPlugin({filename: 'views/home.html', template: 'views/home.html', chunks: ['common', 'Home'], hash: true}),
			new HtmlWebpackPlugin({filename: 'views/detail.html', template: 'views/detail.html', chunks: ['common', 'Detail'], hash: true})
		],
	
		module: {
			loaders: [
				{
					test: /\.js$/, loader: 'babel-loader', // ES6
					exclude: /(node_modules|bower_components|ppaweb\\libs\\webpack)/
				},
				// CSS,LESS打包进JS
				{ test: /\.css$/, loader: 'style-loader!css-loader' },
				{ test: /\.less$/, loader: 'style-loader!css-loader!less-loader' }, // use ! to chain loaders
				// CSS,LESS单独打包
				//{ test: /\.css$/, loader: ExtractTextPlugin.extract("style-loader", "css-loader") },
				//{ test: /\.less$/, loader: ExtractTextPlugin.extract('style-loader', 'css-loader!less-loader') },
		
				{ test: /\.tpl$/, loader: 'ejs'}, // artTemplate/ejs 's tpl
				{
					test: /\.(png|jpg|gif)$/,
					loader: 'url-loader',
					query: {
						name: '[path][name].[ext]?[hash:8]',
						limit: 8192 // inline base64 URLs for <=8k images, direct URLs for the rest
					}
				}
			]
		},
		resolve: {
			alias: {
				'lib0': '../../../ppaweb/libs/webpack', // 从module调用webpack上的公共lib库路径简写
				'lib1': '../../../../ppaweb/libs/webpack', // 从module的子文件夹调用webpack上的公共lib库路径简写
				'lib2': '../../../../../ppaweb/libs/webpack' // 从module的两层子文件夹调用webpack上的公共lib库路径简写
			},
			// 现在可以写 require('file') 代替 require('file.coffee')
			extensions: ['', '.js', '.json', '.coffee']
		}
	};

具体可以参考：[webpack-demo的配置项](https://github.com/diamont1001/webpack-demo/blob/master/example1/webpack.config.js)

## Webpack常用功能 ##
### JS里:CSS及图片引用 ###

	require('./bootstrap.css');
	require('./myapp.less');
	
	var img = document.createElement('img');
	img.src = require('./glyph.png');

- Synchronous
- CSS和LESS会被打包到JS
- 图片可能被转化成 base64 格式的 dataUrl

```
module: {
	loaders: [
		//图片文件使用 url-loader 来处理，小于8kb的直接转为base64
		{ test: /\.(png|jpg)$/, loader: 'url-loader?limit=8192'}
	]
}
```

### LESS/CSS里:图片引用 ###

	background-image: url("./logo.png");

根据配置“url-loader?limit=xxx”来决定把图片转换成base64还是图片链接形式引用。

	module: {
		loaders: [
			//图片文件使用 url-loader 来处理，小于8kb的直接转为base64
			{ test: /\.(png|jpg)$/, loader: 'url-loader?limit=8192'}
		]
	}

### LESS/CSS里：@import 路径问题 ###
LESS里可以通过`@import mixin.less`进行模块化开发，可以在import的路径前面加上~，表示路径以模块处理，支持alias。

`tnpm i @ali/pp-libs --save-dev`

```
# index.less
@import '@ali/pp-libs/libs/base/reset.less';
```

### CSS能单独打包 ###
有时候可能希望项目的样式能不要被打包到脚本中，而是独立出来作为.css，然后在页面中以<link>标签引入。这时候我们需要 `extract-text-webpack-plugin` 来帮忙。

只需两步：

1. 插件安装

`npm install extract-text-webpack-plugin --save-dev`

2. 配置文件webpack.config.js

```
var ExtractTextPlugin = require("extract-text-webpack-plugin");

……

plugins: [
	// 目标文件名规则[name].css
	new ExtractTextPlugin('[name].css', {allChunks: true})
],
module: {
	loaders: [
		{ test: /\.css$/, loader: ExtractTextPlugin.extract("style-loader", "css-loader") },
		{ test: /\.less$/, loader: ExtractTextPlugin.extract('style-loader', 'css-loader!less-loader') },
	]
},
```

### 公共代码自动抽离 ###
提取多个页面之间的公共模块，并将该模块打包为 common.js

`A.js, B.js => a.js, b.js, common.js`

    // 分析以下模块的共用代码, 单独打一个包到common.js
	var commonsPlugin = new webpack.optimize.CommonsChunkPlugin(/*chunkName=*/'common', /*filename=*/'common.js');
	
	plugins: [
		commonsPlugin
	],

记得要在HTML手动引入common.js

### 自定义公共模块提取 ###
上面是自动在所有入口的js中提取公共代码，并打包为common.js。

有时候我们希望能更加个性化一些，比如我希望:

`A.js+C.js => AC-common.js`

`B.js+D.js => BD-common.js`

我们可以这样配：

```
module.exports = {
	entry: {
		A: "./a.js",
		B: "./b.js",
		C: "./c.js",
		D: "./d.js",
		E: "./e.js"
	},
	output: {
		filename: "[name].js"
		},
	plugins: [
		new CommonsChunkPlugin("AC-commons.js", ["A", "C"]),
		new CommonsChunkPlugin("BD-commons.js", ["B", "D"])
	]
};
// <script>s required:
// a.html: AC-commons.js, A.js
// b.html: BD-commons.js, B.js
// c.html: AC-commons.js, C.js
// d.html: BD-commons.js, D.js
// e.html: E.js
```

### HTML自动引用 JS/CSS ###
有时候我们连HTML里的JS/CSS资源都懒的写，也是可行的，HTML也可以当成模块来写。

`npm install html-webpack-plugin --save-dev`

```
var HtmlWebpackPlugin = require('html-webpack-plugin'); // Html文件处理

module.exports = {

	……

	plugins: [
		/**
		* HTML文件编译，自动引用JS/CSS
		* 
		* filename - 输出文件名，相对路径output.path
		* template - HTML模板，相对配置文件目录
		* chunks - 只包含指定的文件（打包后输出的JS/CSS）,不指定的话，它会包含生成的所有js和css文件
		* excludeChunks - 排除指定的文件（打包后输出的JS/CSS），比如：excludeChunks: ['dev-helper']
		* hash
		*/
		new HtmlWebpackPlugin({filename: 'views/list.html', template: 'src/modules/app/list/index.html', chunks: ['common', 'List'], hash: true}),
		new HtmlWebpackPlugin({filename: 'views/detail.html', template: 'src/modules/app/detail/index.html', chunks: ['common', 'Detail'], hash: true})
	],
};
```

具体参考 [webpack-demo的配置项](https://github.com/diamont1001/webpack-demo/blob/master/example1/webpack.config.js)

### 全局变量 ###
有些代码我们只想在开发环境使用（比如log），这里，我们需要用到全局变量插件：webpack.DefinePlugin

```
module.exports = {
	plugins: [
		// 全局变量
		new webpack.DefinePlugin({
			// __DEV__: JSON.stringify(JSON.parse(process.env.DEBUG || 'false')), //通过环境变量设置
			__DEV__: JSON.stringify(JSON.parse('true')), // 开发调试时把它改为true
			__HELLO__: JSON.stringify('hello world')
		})
	]
};
```

js中调用

```
if(__DEV__) {
	console.log(__HELLO__);
}
```

注意：webpack -p 会执行 uglify dead-code elimination, 任何这种代码都会被剔除, 所以你不用担心秘密功能泄漏.

### 异步加载 ###

`require.ensure`

语法：
```
require.ensure(dependencies: String[],
		callback: function([require]),
		[chunkName: String])
```

与require AMD类似，也是在需要的时候才会加载相应的模块。但不同的是，require.ensure在模块被下载下来后（模块还没被执行）便立即执行回调函数.
另外require.ensure可以指定构建后chunk名，如果之前已有require.ensure指定了该名称，webpack会将这些模块统一合并到一个模块集里。

简单例子

	// 异步加载
	if(i < 0) {
		require.ensure([], function() {
			require('a.js');
		});
	}

定义异步加载文件名字(webpack.config.js)

	output: {
		chunkFilename: "[id].chunk.[hash:8].js"
	},

生成的异步文件引用逻辑自动包含在源目标JS中，不用手动引用，所以以上文件名随便怎么定义都不影响。

![](https://github.com/diamont1001/webpack-summary/blob/master/imgs/require.ensure-demo.png?raw=true)

### file-loader ###
图片加载器url-loader其实是对file-loader的一个封装

```
loaders: [
	{
		test: /\.(png|jpg|gif)$/,
		loader: 'url-loader',
		query: {
			name: '[path][name].[ext]?[hash:8]',
			limit: 8192
		}
	}
]
```

如果文件超出体积, 就给一个这样规则的文件名
![](https://github.com/diamont1001/webpack-summary/blob/master/imgs/file-loader-demo.png?raw=true)

参考：[https://github.com/webpack/file-loader](https://github.com/webpack/file-loader)

### ES6支持 ###

	module: {
		loaders: [
			{
				test: /\.js$/, loader: 'babel-loader', // ES6
				exclude: /(node_modules|bower_components|ppaweb\\libs\\webpack)/
			},
		]
	},

参考：[http://npm.taobao.org/package/babel-loader](http://npm.taobao.org/package/babel-loader)

### Alias:项目迁移更方便 ###
webpack允许配置路径的别名，这样在一些外部资源的依赖的时候显得格外有用，对以后的项目迁移等都起到不小的作用。

	resolve: {
		alias: {
			// 从module调用公共libs上的库路径简写
			'lib0': '../../../libs',
	
			// 从module的子文件夹调用公共libs上的库路径简写
			'lib1': '../../../../libs', 
	
			// 从module的两层子文件夹调用公共libs上的库路径简写
			'lib2': '../../../../../libs' 
		}
	}

```
# module/index.js
require('lib0/proxy');

# module/app/index.js
require('lib1/proxy');

# module/app/header/index.js
require('lib2/proxy');
```

### shimming ###
在 AMD/CMD 中，我们需要对不符合规范的模块（比如一些直接返回全局变量的插件）进行 shim 处理，这时候我们需要使用 [exports-loader](https://github.com/webpack/exports-loader) 来帮忙：

	{ test: require.resolve("./src/js/tool/swipe.js"),  loader: "exports?swipe"}

之后在脚本中需要引用该模块的时候，这么简单地来使用就可以了：

	require('./tool/swipe.js');
	swipe(); 

## Webpack模块编写 ##
### 模块框架 ###

	// var $ = require('zepto');
	// require('./index.less');

	!(function () {
	
		var module1 = (function () {
			var _e = {};
		
			_e.test = function () {
				// do something here
			};
		
			return _e;
		})();

		window.module1 = module1;
		
		try {
			module.exports = module1;
		} catch (e) {}

	})();

## 旧项目迁移方案 ##
### 1. 入口文件 ###
一般一个页面(HTML)对应一个入口文件

	/views/a.html
	/views/b.html
	/views/c.html

	entry: {
		A: 'modules/app/a.js',
		B: 'modules/app/b.js',
		C: 'modules/app/c.js'
	}

### 2. 文件引用 ###
![](https://github.com/diamont1001/webpack-summary/blob/master/imgs/1.png?raw=true)

![](https://github.com/diamont1001/webpack-summary/blob/master/imgs/2.png?raw=true)

### 3. 优化 ###
- common.js
- css单独打包
- 异步加载
- HTML模板（html-webpack-plugin）

## 附录 ##
- [Git官网](https://webpack.github.io/)
- [入门指引（含例子webpack+react）](https://github.com/petehunt/webpack-howto)
- [webpack简易教程PPT](http://slides.com/diamont1001/webpack)
- [webpack开发原型demo](https://github.com/diamont1001/webpack-demo)

## Q&A ##
###Q. HTML里引用JS能自动生成访问后缀吗？比如a.js?2016###
A. 插件html-webpack-plugin

