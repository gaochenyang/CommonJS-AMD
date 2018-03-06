 # JavaScript模块规范(CommonJS,AMD)

在JavaScript模块一文中介绍了如何组织代码实现模块化。模块化能隐藏私有的属性和方法，只暴露出公共接口。这样别人就不需要从头开始造轮子，直接用你的模块中定义的功能就行了。而且保证了命名空间，不会出现命名冲突。

但如果没有一套规范做参照，每个人都随自己的喜好定义模块，使用别人的模块就会出现障碍。本篇就介绍一下通用的定义JS模块的规范：CommonJS和AMD

 # # CommonJS
 
 Nodejs的模块系统就采用CommonJS模式。CommonJS标准规定，一个单独的文件就是一个模块，模块内将需要对外暴露的变量放到exports对象里，可以是任意对象，函数，数组等，未放到exports对象里的都是私有的。用require方法加载模块，即读取模块文件获得exports对象。

例如定义个hi.js：

```
var str = 'Hi';

function sayHi(name) {
  console.log(str + ', ' + name + '!');
}

module.exports = sayHi;
```
在hi模块中定义了sayHi函数，用exports将它暴露出去。未被暴露出去的变量str是无法被外部访问的。其他模块要用这个函数的话，需要先require这个hi模块：

```
var Hi = require('./hi');
Hi('Jack');     // Hi, Jack!
```
第一行里的变量Hi，其实就是hi.js里用module.exports = sayHi;输出的sayHi函数。

注意上面用require加载时写的是相对路径，让Nodejs去指定路径下加载模块。如果省略相对路径，默认就会在node_modules文件夹下找hi模块，那很可能因为找不到而报错。如果你加载的是Node内置模块，或npm下载安装后的模块，可以省略相对路径，例如：

```
var net = require('net');
var http = require('http');
```
在JavaScript模块一文中介绍过JS模块的写法，这里出现的module，exports，require是JS的新语法吗？不是新语法，只是CommonJS的语法糖。Node会将上述hi.js编译成：
```
// 准备module对象:
var module = {
    id: 'hi',
    exports: {}
};
var load = function (module) {
    function sayHi(name) {
        console.log('Hi, ' + name + '!');
    }

    module.exports = sayHi;
    return module.exports;
};
var exported = load(module);
// 保存module:
save(module, exported);
```
上来先定义一个module对象，属性id是该模块名即文件名，属性exports就是最终需要暴露出来的对象。因此我们在hi.js代码里明明没有var声明过module对象，但module.exports = sayHi;居然不会报错。原因就是module是Node在加载js文件前已经事先替我们准备好了。最后Node会用自定义函数save将这个module对象保存起来。

Node保存了所有的module对象后，当我们用require()获取module时，Node会根据module.id找到对应的module，并返回module. exports，这样就实现了模块的输出。

CommonJS是同步的，意味着你想调用模块里的方法，必须先用require加载模块。这对服务器端的Nodejs来说不是问题，因为模块的JS文件都在本地硬盘上，CPU的读取时间非常快，同步不是问题。

但在客户端浏览器用CommonJS加载模块将取决于网速，如果采用同步，网络情绪不稳定时，页面可能卡住。因此针对客户端出现了AMD异步模块定义。

# #AMD
AMD（Asynchronous Module Definition）异步加载模块。AMD标准规定，用define来定义模块，用require来加载模块：
```
define(id, [depends], factory);  
require([module], callback);
```
先看define定义模块的示例：

```
define(['module1', 'module2'], function (module1, module2) {
    ……
    return { … };
});
```
第一个参数id是你的模块名，上例省略。事实上这个id没什么用，就是给开发者看的。

第二个参数[depends]是该模块所依赖的其他模块，上例中该模块依赖另两个模块module1和module2。如果你定义的模块不依赖其他任何模块，该参数可以省略。

第三个参数factory，生产出（即return出）一个对象供外部使用（我看到有的资料里将该参数命名为callback，感觉callback语义方面有点暧昧，命名成factory更贴切）。

定义好模块后，再看用require加载模块的示例：

```
require(['yourModule1', 'yourModule2'], function (yourModule1, yourModule2) {
    ……
}); 
```
可以和CommonJS的require方法对比一下，AMD的require多了一个参数callback。

第一个参数是需要加载的模块名，可以是一个数组，意思是加载多个模块。加载模块时，如果该模块的define里有[depends]参数，就会先加载[depends]里指定的依赖模块。加载完[depends]里的依赖模块后，运行define里的factory方法得到该模块的实例。然后将该实例依次传入第二参数callback作为参数。

第二个参数callback回调函数，参数是根据第一个参数，依次加载模块后得到的实例对象。等第一个参数指定的模块全部加载完后，会执行该callback。

require加载过程分几步：

第一步，将依赖列表的模块名转换为URL，常见的是basePath + moduleID + ".js"。例如：

```
require(["aaa", "bbb"], function(a, b){});
//将在require内部被转换成：
//require(["http://1.2.3.4/aaa.js", 
//       "http://1.2.3.4/bbb.js"], function(a, b){});

```
第二步，从检测对象数组里查看该模块是否被加载过（状态是否为2），或正在被加载（状态是否为1）。只有从来没加载过此节点，才会进入加载流程。

第三步，将该模块的状态设为1，表示正在被加载。创建script标签，模块的URL添加到src里，并绑定onload，onreadystatechange，onerror事件，将script插入DOM树中开始加载。浏览器加载完后会触发绑定的on事件，里面可以将模块的状态改为2，表明已经加载完毕。

第四步，将模块的URL，依赖列表，状态等构建一个检测对象数组，供第二步检测用。

你可以将所有需要用到模块里变量和方法的代码，都放到callback中。require下面的代码将继续执行，这样就能避免浏览器卡住假死的情况。

实现了AMD规范的JS库有：require.js。看一个用require.js的例子：

```
//myModule1.js
define(function() {
    var m1 = {};
    m1.say = function() {
        console.log('Hi myModule1!');
    }
    return m1;
});

//myModule2.js
define(['myModule3'], function(m3) {
    var m2 = {};
    m2.say = function() {
        m3.say();
        console.log('Hi myModule2!');
    }
    return m2;
});

//myModule3.js
define(function() {
    var m3 = {};
    m3.say = function() {
        console.log('Hi myModule3!');
    }
    return m3;
});

//HTML
console.log("before require");
require(['myModule1','myModule2'], function(m1, m2){         
    m1.say();
    m2.say();
});
console.log("after require");
//before require
//after require
//Hi myModule1!
//Hi myModule3!
//Hi myModule2!

```

HTML中先执行console.log(“before require”);，打印出第一条。

然后执行require，因为require.js是AMD异步加载，所以执行require后的console.log(“after require”);语句，打印出第二条。

执行require依次加载模块。先加载myModule1。发现myModule1的define里无[depends]参数，不依赖其他模块。因此运行define里的factory方法获得myModule1的实例对象m1。

再加载myModule2。发现myModule2依赖myModule3，因此加载myModule3。myModule3无[depends]参数，不依赖其他模块。因此运行factory获得myModule3的实例对象m3。

加载完myModule3后，运行myModule2的factory获得myModule2的实例对象m2。

myModule2也加载完毕后，require的所有模块均加载完毕，运行回调函数。前面获得的实例对象m1和m2作为参数传给回调函数。

运行m1.say();打印出第三条

运行m2.say();，根据方法定义，先运行m3.say();打印出第四条，再运行console.log(‘Hi myModule2!’);打印出第五条。

有些库并没有严格遵照AMD，用define()函数定义的模块，此时需要用shim技术（所谓shim就是垫片，即把一个新的API引入到一个旧环境中，并且仅靠旧环境中已有的手段来实现）。例如require.js就为require.config定义了shim属性，我们用它来加载并没有采用AMD规范的underscore和backbone这两个库：

```
require.config({
    shim: {
        'underscore':{
            exports: '_'
        },
        'backbone': {
            deps: ['underscore', 'jquery'],
            exports: 'Backbone'
        }
    }
});

```
# 总结
无论是被Nodejs采用的服务器端的CommonJS规范，还是客户端的AMD规范，都可以将我们自定义的模块规范化，便于供他人使用。
