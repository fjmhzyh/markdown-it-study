

# Markdown-it源码分析



## 基础用法

```javascript
const MarkdownIt = require('markdown-it');

const md = new MarkdownIt();
var htmlString = md.render('## hello!');
```



## 解析流程 - render方法做了什么

要理解`MarkdownIt`的解析流程,就必须知道render方法做了哪些工作.  首先让我们来看一下`render`方法的源码

```javascript
MarkdownIt.prototype.render = function(src, env) {
  env = env || {};
  return this.renderer.render(this.parse(src, env), this.options, env);
};

MarkdownIt.prototype.parse = function(src, env) {
  if (typeof src !== "string") {
    throw new Error("Input data should be a String");
  }
  var state = new this.core.State(src, this, env);
  this.core.process(state);
  return state.tokens;
};
```

稍作整理, 我们可以将以上代码简化为

```javascript
MarkdownIt.prototype.render = function(src, env) {
  env = env || {};
  var tokens = this.parse(src, env);
  var htmlString = this.renderer.render(tokens, this.options, env);
  return htmlString;
};
```

根据源码我们可以将整个解析流程归纳成两个步骤:

1. 将传入的`markdown字符串`解析成`tokens`
2. 将`tokens`转化成`HTML字符串`, 并返回给用户



## Tokens - 转换的秘诀

现在我们接触到了`MarkdownIt`的第一个核心概念 - `tokens`. 那么什么是tokens呢? 我们通过控制台打印可以获得以下结果

```javascript
var tokens = [
  [
    type: "heading_open",
    tag: "h2",
    attrs: ["id", "tokens2"],
    block: true,
    content: "",
    children: null,
  ],
  [
    type: "inline",
    tag: "h2",
    attrs: null,
    block: false,
    content: "hello!"
    children: [
      [
        type: "text"        
        tag: ""
        attrs: null
        block: false
        children: null
        content: "Tokens2"
      ]
    ]
  ],
  [
    type: "heading_close",
    tag: "h2",
    attrs: null,
    block: true,
    content: "",
    children: null,
    hidden: false,
  ]
]
```

可以看到, 一个`token`包含了`type`,  `tag`,   `content`, `children`等属性.  其中`tag`代表了`token`将被渲染成的`HTML`标签. `content`则表示内容.`children`则是该标签的子元素.

用户输入的`markdown字符串`, 在经过`md.parse`方法处理之后,都会转换成一个个的`token`. 并放到`state.tokens`数组中.

**所以我们可以将`tokens`理解为从`markdown`转换到`html`的一种必经的, 具有固定格式的中间态**



## Ruler - token是怎么来的

知道了`tokens`是做什么的,  那么`tokens`是怎么来的呢?  现在让我们一起来学习`MarkdownIt`的第二个核心概念 - `Ruler`.

 让我们一起看看`md.parse`方法做了什么

`Ruler`的构造方法非常简单. `__rules__`数组, 用于缓存各种解析规则, 也就是`rule`. 根据作者的注释, 我们也能大致了解rule大概长什么样, 包含了`name`,`enabled`, `fn`, `alt`4个属性. `__cache__`则 缓存了这些规则.

```javascript
function Ruler() {
  // List of added rules. Each element is:
  //
  // {
  //   name: XXX,
  //   enabled: Boolean,
  //   fn: Function(),
  //   alt: [ name2, name3 ]
  // }
  //
  this.__rules__ = [];
  // Cached rule chains.
  this.__cache__ = null;
}
```

看完了`Ruler`, 我们再来看看 具体的`rule`, 也就是解析规则. rule的作用可以归纳为以下两点

1. 标准化或格式化`markdown字符串`, 使其具备特定的格式, 比如`normalize`函数
2. 生成`token`, 比如`block`函数.  所有的rules将按照特定的顺序对`markdown字符串`进行处理, 并最终生成`tokens`数组

```javascript
function normalize(state) {
  var str;
  // Normalize newlines
  str = state.src.replace(NEWLINES_RE, "\n");
  // Replace NULL characters
  str = str.replace(NULL_RE, "\ufffd");
  state.src = str;
};


function block(state) {
  var token;

  if (state.inlineMode) {
    token          = new state.Token('inline', '', 0);
    token.content  = state.src;
    token.map      = [ 0, 1 ];
    token.children = [];
    state.tokens.push(token);
  } else {
    state.md.block.parse(state.src, state.md, state.env, state.tokens);
  }
};
```



## Renderer - token的解析

知道了`tokens`是怎么来的, 以及它的作用, 我们再来看看`tokens`是如何被解析成`html`的.

回到`md.render`方法, `tokens`生成之后, 作为参数, 传递给了`md.renderer.render`方法. 先来看看`Renderer`对象,  Renderer对象缓存了一系列默认的rules.

```javascript
MarkdownIt.prototype.render = function(src, env) {
  env = env || {};
  var tokens = this.parse(src, env);
  var htmlString = this.renderer.render(tokens, this.options, env);
  return htmlString;
};


// Renderer构造方法
function Renderer() {
  this.rules = assign({}, default_rules);
}
```



再来看看`Renderer.render`方法. 可以看到传入的`token`根据不同的`type`被传递到不同的`rule`进行了处理.

 **这里的`rule`不同于`Ruler`对象保存的`rule`. 它的作用只有一个 - 将`token`解析成`html`**

```javascript
Renderer.prototype.render = function(tokens, options, env) {
  var i, len, type, result = "", rules = this.rules;
  for (i = 0, len = tokens.length; i < len; i++) {
    type = tokens[i].type;
    if (type === "inline") {
      result += this.renderInline(tokens[i].children, options, env);
    } else if (typeof rules[type] !== "undefined") {
      result += rules[tokens[i].type](tokens, i, options, env, this);
    } else {
      result += this.renderToken(tokens, i, options, env);
    }
  }
  return result;
};
```



下面是一个内置的rule, 用来解析`type`为`code_inline`的token, 并最终返回一串`html`.

```javascript
var default_rules = {};

default_rules.code_inline = function (tokens, idx, options, env, slf) {
  var token = tokens[idx];

  return  '<code' + slf.renderAttrs(token) + '>' +
          escapeHtml(tokens[idx].content) +
          '</code>';
};
```



## 总结

通过简单的分析, 我们可以将`MarkdownIt`的执行过程归纳为下图, 希望能让你对`MarkdownIt`有一个更好的理解.
