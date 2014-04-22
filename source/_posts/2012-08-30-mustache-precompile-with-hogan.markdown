title: "Mustache.js/Hogan.js 模板预编译"
date: 2012-08-30 16:18
tags: [Frontend, Mustache, Hogan]
---

[mustache.js](https://github.com/janl/mustache.js) 是[粉笔网](http://fenbi.com)用的一个开源前端模板引擎，无逻辑的设计，简单好用，性能也不错。

``` javascript 一个简单的 mustache.js 渲染例子 demo.js
var template = "hello {name}}!"; // 因为代码高亮插件的 bug，这里 name 左边少了一个 {，实际代码中要加上
console.log(Mustache.render(template, {name: "foo"})); // hello foo!
console.log(Mustache.render(template, {name: "bar"})); // hello bar!
```

Mustache 在 render 一个模板时，首先会将这个模板编译成一个模板函数。比如上面例子里的 `hello {{ "{{name" }}}}` 模板，会被编译成一个模板函数：

``` javascript
function anonymous(c, r) {
    return "" + "hello\u0020" + r._name("name", c, true) + "\u0021";
}
```

大规模应用时，模板的编译过程会花掉整个 render 过程中 30% 左右的时间。

## 使用 Hogan.js 预编译 Mustache 模板

Mustache 模板的这个问题已经被不少人遇到，也有很多解决办法。比如 twitter 发布的 [Hogan.js](http://twitter.github.com/hogan.js)。Hogan.js 是 Mustache 模板引擎的另一套实现，增加了预编译机制，使得模板字符串可以在打包阶段被预先处理成模板函数，这样浏览器就不必再重复去编译模板。

Hogan.js 同时提供了可以运行与浏览器端和 node.js 环境下的代码，node.js 负责打包时预编译，浏览器端负责用预编译后的代码渲染页面。

首先通过 npm 安装 hogan.js 的 node.js 环境：

```
$ npm install hogan.js
```

然后对源代码进行一些修改，在模板字符串的旁边加上一些标记，让打包脚本可以找到模板字符串：
``` javascript 修改后的例子 demo.js
var Template = {
    _cache: {},

    // 所有的模板放在这个对象下
    _template: {
        hello: /*TMPL*/"hello {name}}!"/*TMPL*/ // 因为代码高亮插件的 bug，这里 name 左边少了一个 {，实际代码中要加上
    },

    // 这个适配函数会同时处理字符串模板和模板函数的情况
    render: function (name, data) {
        if (!this._cache[name]) {
            // 如果代码被预编译过，则不需要 compile
            if (typeof this._template[name] === 'function') {
                this._cache[name] = new Hogan.Template(this._template[name]);
            } else if (typeof this._template[name] === 'string') {
                this._cache[name] = Hogan.compile(this._template[name]);
            }
        }

        return this._cache[name].render(data);
    }
};

console.log(Template.render('hello', {name: "foo"})); // hello foo!
console.log(Template.render('hello', {name: "bar"})); // hello bar!
```

``` javascript nodejs 环境中的预编译过程
var hogan = require("hogan.js");
var fs = require("fs");
var fileContent = fs.readFileSync("demo.js", "utf-8");
fileContent.replace(/\/\*TMPL\*\/"(.*?)"\/\*TMPL\*\//g, function ($0, $1) {
    return hogan.compile($1, {
        asString: true
    });
});
fs.writeFileSync("demo.js", fileContent, "utf-8");
```

源代码编译完之后，模板字符串就变成了模板函数：
``` javascript
/* ... */
    hello: function(c,p,i){var _=this;_.b(i=i||"");_.b("hello ");_.b(_.v(_.f("name",c,p,0)));_.b("!");return _.fl();;}
/* ... */

console.log(Template.render('hello', {name: "foo"})); // hello foo!
console.log(Template.render('hello', {name: "bar"})); // hello bar!
```

## 参考资料

* [Precompiling hogan.js/mustache templates on a Java server with Struts 2](http://www.grobmeier.de/precompiling-hogan-jsmustache-templates-on-a-java-server-with-struts-2-16012012.html#.UD82lmhiivI)
