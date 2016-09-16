# Vue learing notes

My note of learning Vue.js

## 0.1.0

#### 初始设置 

把依赖啥的都安装好，`component` 这个全局库**不能**使用最新版本否则执行`grunt task`时会报错 

```
$ npm i -g grunt 
$ npm i -g component@0.16.3
$ npm i
$ component install component/emitter
$ grunt (--force)

```

---


#### initial setup [83fac01](https://github.com/vuejs/vue/tree/83fac017f96f34c92c3578796a7ddb443d4e1f17)

`Vue` 最初的名字叫 `element` 。
    
---
    
#### rename [871ed91](https://github.com/vuejs/vue/tree/871ed9126639c9128c18bb2f19e6afd42c0c5ad9)

改名叫 `seed` ，初期思路的尝试。
    
对`Mustache`语法 `{{}}` 做一个replace,替换掉原来的innerHTML

```html
<p>{{msg}}</p>
// parse后：
<p><span data-element-binding="msg"></span></p>
```

同时维护一个 `bindings` 变量，存储所有 `{{}}` 中的绑定变量，如 `msg`

对 `msg` 维护一个`els`的属性，使用 `document.queryselectAll("[data-element-binding=*]")`  拿到所有DOM，存入。然后删除 `data-element-binding` 这个属性。

使用 `Object.defineProperty` 中的 `set` 钩子完成`值 -> dom` 的**单向绑定**。

```js
Object.defineProperty(data, variable, {
    set: function (newVal) {
        // 赋值时设定binding, 改动dom
        [].forEach.call(bindings[variable].els, function (e) {
            bindings[variable].value = e.textContent = newVal
        })
    },
    get: function () {
        return bindings[variable].value
    }
})
```
    
---

- naive implementation [a5e27b1](https://github.com/vuejs/vue/commit/a5e27b1174e9196dcc9dbb0becc487275ea2e84c)
 
初期的实现，作者当时ms在参考 [Rivets.js](https://github.com/mikeric/rivets) 

**filters.js**

`filters` 来对绑定变量做特殊处理，这里仅是一个 `capitalize` 。

**directive.js**

`directives` 一些内置指令，是 `vue` 使用者很熟悉的指令：
- text: el.textContent
- show: el.style.display
- class: add/remove className
- on: update, unbind, customFilter 

**main.js**

`new Seed(opts)`做初始化，参数`opts`是根dom。

在根dom里面把为具有`sd`的`prefix`，且满足在当前 `内置directives` 中条件的dom 找出来。比如 

```html
<div id="test" sd-on-click="changeMessage | .button">
    <p sd-text="msg | capitalize"></p>
    <p sd-show="something">YOYOYO</p>
    <p class="button" sd-text="msg"></p>
    <p sd-class-red="error" sd-text="hello"></p>
</div>
```

整个流程如下
    
    
```
graph TB
A(取出所有内置指令)-->B(获取root);
B-->C(解析并parse该内置指令,设置update);
C-->D(在seed scope中绑定所有key);
D-->E(对所有key做get/set处理)

```


同时，在 `set` 函数执行时, 由于内置指令的预定义函数绑定了directive的update。(见函数`bindAccessors`)，
所以统一调用 `directive.update` 方法。
不同的内置指令对应不同的行为，如：

```js
{
    text: function (el, value) {
        el.textContent = value || ''
    },
    show: function (el, value) {
        el.style.display = value ? '' : 'none'
    }
}
```

最后，从`TODO`中可以了解作者的思路：

- nested levels by parsing dot syntax (which means nested getter/setters...)

例如在传

```js
new Seed(dom, {
    'msg.wow': 'hello'
})
```

的情况下，即有nested getter/setter

- repeat directive by watching an array： 这个应该是 `v-for` 之类的遍历数组了
- parse textNodes：对TextNode进行特殊处理，比如{{hello}},转化成`dom attribute`效率还是比较低
- make Seeds compositable：可以mixin?
- formatter arguments: 不知道要干嘛
- Seed.extend(): 创建一个新实例
- options to pass in templates to Seed.create()：new Seed(arguments)

    
---

#### filter value should not be written [3eb7f6f](https://github.com/vuejs/vue/commit/3eb7f6fc72f40ac43aa502e2e8cf70404a490b11)

#### dump, destroy, fix filters [ec39439](https://github.com/vuejs/vue/commit/ec394395569ea81e50209b9e241e789cfacde588)

修了个bug,就是用户原始传入的值被`filter`处理后，是需要保持原值的，`filter`处理的是值的副本。

`Seed`构造函数加入 `dump` 和 `destroy`

---


#### refrator [cf1732b](https://github.com/vuejs/vue/commit/cf1732bea21dcc1637d587d295d534535a92d2b7)

把内置指令的parse/具体方法划分为 `directive.js`和 `directives.js`

做了 `this` 的绑定

`class`和 `repeat` 暂时还没有实现 

---

#### augmentArray seems to work [154861f](https://github.com/vuejs/vue/commit/cf1732bea21dcc1637d587d295d534535a92d2b7)

覆写了`Array.prototype.push`方法，push时进行mutate,做到**数组长度改变时的监听**

```js
function augmentArray (collection, directive) {
    collection.push = function (element) {
        push.call(this, arguments)
        directive.mutate({
            event: 'push',
            elements: slice.call(arguments),
            collection: collection
        })
    }
}
```

再看这一段

```
<div id="test" sd-on-click="changeMessage | delegate .button">
...
</div>
```

filter.js里面使用了`Element.prototype.matches` 来进行`事件代理` ，但是这里有bug, `e.target !== e.currentTarget` 时需要往`parentNode`上继续检测的嘛，~~而且`matches`有兼容性问题，待后续看解决方案是`polyfill`还是其他。~~

```js
delegate: function (handler, selectors) {
        return function (e) {
            var match = selectors.every(function (selector) {
                return e.target.webkitMatchesSelector(selector)
            })
            if (match) handler.apply(this, arguments)
        }
    }
```

PS: `Element.matches()` 支持情况, 原则上是可以`加前缀通用`的， 基本都已经支持。即使作`polyfill`也没啥。



Feature | Chrome | Firefox (Gecko) | Internet Explorer | Opera | Safari (WebKit) 
---|---|---|---|---|---
Original support with a non-standard name | Yes | 3.6 (1.9.2) | 9.0 | 11.5 / 15.0 | 5.0 |
Specified version | 34 | 34 | ? | 21.0 | 7.1 |

此数据来自 [mozilla developer](https://developer.mozilla.org/en-US/docs/Web/API/Element/matches)


---

#### WIP [79760c0](https://github.com/vuejs/vue/commit/79760c09d50caca7ca27cd85991eb2c6e9ba3231)

抽离设置到`config.js`
`filter.js` 当filter函数没定义时throw Error

数组的监听方面，监听了几个常用的方法, 做本职工作的同时加入`callback`

```js
var proto = Array.prototype,
    slice = proto.slice,
    mutatorMethods = [
        'pop',
        'push',
        'reverse',
        'shift',
        'unshift',
        'splice',
        'sort'
    ]

module.exports = function (arr, callback) {

    mutatorMethods.forEach(function (method) {
        arr[method] = function () {
            proto[method].apply(this, arguments)
            callback({
                event: method,
                args: slice.call(arguments),
                array: arr
            })
        }
    })

}
```

`main.js` 只作为一个入口点，主要逻辑移到了`seed.js`


---

#### refractor [5ce3b82](https://github.com/vuejs/vue/commit/5ce3b82b91a134f57dd1dffe8553cec369a56c70)

`sd-repeat` --> `sd-each`

引入`Seed.extend` 替换原来的 `Seed.seed`（ms原来这种命名感觉会绕晕），通过原型继承创建子组件

作者打算写一个`todo` , 毕竟是MVVM的标准Demo...

---

#### kinda working now [952ab43](https://github.com/vuejs/vue/commit/952ab433a5a37f56ce95b06578b028ecdfdcad6c)

作者在`todo.md`里面做的思考：

- ? separate scope data and prototype methods // think about this

`MVVM` 里面不可避免的会碰上 `nested scope` 。 如： 

```html
<parent>
  <child>
    <button sd-on-click="change" sd-text="hello">
  </child>
</parent>
```

那么问题是：比如parent 和 child 都有`change`方法，data中都有`hello`，应该用哪个？遍历dom树解析的时候如何解析？

假设我们希望`hello`是只读取`child`的,比较合情理。 

那么有N个`child` 绑定 `change`事件呢？我们又希望能够做`delegate`绑到`parent`上去.....

那么如何实现这样的功能，甚至是一个 `isolated scope` ？

这个任何一个`MVVM`框架都会做而且必须做。

---

#### sd-each-* works [3149839](https://github.com/vuejs/vue/commit/31498397366dc036911690e06670a1b0d1746654)

实现了数组循环绑定。

```
<li sd-each-todo="todos">
    <span class="todo" sd-text="todo.title" sd-class-done="todo.done"></span>   
</li>
```

要实现循环绑定，那么按上面例子就需要插入n个span. 这里引入了一个`childSeed` ,当长度大于0时进行插入：

```js
function buildItem (data, index, collection) {
    var node = this.el.cloneNode(true),
        spore = new Seed(node, data, {
            eachPrefixRE: this.prefixRE,
            parentScope: this.seed.scope
        })
    this.container.insertBefore(node, this.marker)
    collection[index] = spore.scope
    return spore
}
```

在`Seed.prototype._compileNode`中对`textnode`进行处理, `_compileTextNode`暂还未实现：

```js
if (node.nodeType === 3) {
    // text node
    self._compileTextNode(node)
}
```

#### big refactor.. again [23a2bde](https://github.com/vuejs/vue/commit/23a2bde88917cd3f043beca4f5da3b37b0f9de29)

重构狂魔的作者又搞出来一个`controller`的概念。。是否也受了ng的影响。。

`controller` 主要就是解决 `scope nesting`的问题，这个版本还没实现

实际就是创建一个`Seed`新实例, 避免混淆

```js
Seed.controller = function (id, extensions) {
    if (controllers[id]) {
        console.warn('controller "' + id + '" was already registered and has been overwritten.')
    }
    var c = controllers[id] = Seed.extend(extensions)
    return c
}
```

原先的 `directive.js` 和 `directives.js` 显然看名字有点晕，所以改成了`binding.js`

前面提到的：

> filter.js里面使用了`Element.prototype.matches` 来进行`事件代理` ，但是这里有bug, `e.target !== e.currentTarget` 时需要往`parentNode`上继续检测的嘛，而且`matches`有兼容性问题，待后续看解决方案是`polyfill`还是其他。

`filter.js`中的`deletegate`方法做了一个`delegateCheck` ，做冒泡遍历，修掉之前bug。

`Seed.bootstrap` 对于`controller`做一个自检测，但是还未加入流程。

---

#### kinda work now except for scope nesting [62b75d4](https://github.com/vuejs/vue/commit/62b75d49838744f8b865064256c5f0f8df51616e)

#### almost there! scope nesting [15ffaa4](https://github.com/vuejs/vue/commit/15ffaa4166f65ae62eb3ac64fa0ca83e1455bda2)

`_compileTextNode` 方法做了实现：先match, 如果自己没有，去找`parentSeed`

```
<ul sd-show="todos">
    <li class="todo"
        sd-controller="Todo"
        sd-each="todo:todos"
        sd-class="done:todo.done"
        sd-on="click:changeMessage, click:todo.toggle"
        sd-text="msg"
    ></li>
</ul>
```

里面对于`nested scope`的实现：

```js
var key = bindingInstance.key,
        epr = this._options.eachPrefixRE,
        isEachKey = epr && epr.test(key),
        scopeOwner = this
    // TODO make scope chain work on nested controllers
    if (isEachKey) {
        key = key.replace(epr, '')
    } else if (epr) {
        scopeOwner = this._options.parentSeed
    }
```

感觉现在这样的设计用起来还是有点囧的...必须要标注一下自己是属于哪个scope的

---

#### finally workssssss [08e7992](https://github.com/vuejs/vue/commit/08e7992c2a7fd0f686d2bf22205822dacf74b53f)

还在解决 `nested scope`的问题。

#### milestone reached, update todo [83665f9](https://github.com/vuejs/vue/commit/83665f9962d041378213bffb61647eff25e67a46)

一个基本的`MVVM`已经实现了。但是原来的`nested scope`实现被作者自己否了。

```js
else if (ctrlExp && !root) { // nested controllers
    // TODO need to be clever here!
} 
```

`todo.md`

- nested controllers - but how to inherit scope? 作者在思考作用域继承的问题。
- improve arrayWatcher 
- parse textNodes 
- computed properties : computed properties的萌芽，天才的想法~

---

#### emitter [1d7a94e](https://github.com/vuejs/vue/commit/1d7a94ed1abfa5273d6071e1f1f1e2825d32b9a3)

用到了[component/emitter](https://github.com/component/emitter), 让它作为`Seed.prototype` 的 `mixin`,
用来做事件传递


`seed.js`中：
```js
Emitter(Seed.prototype)
```

```js
Seed.controller('Todo', function (scope, seed) {
    scope.toggle = function () {
        scope.done = !scope.done
        seed.parentSeed.emit('toggle', seed)
    }
})
```


---

#### todo demo kinda works [a6b7257](https://github.com/vuejs/vue/commit/a6b72570e22f5cfaaf7bc70e4e7a7dbf18ec23c0)

`Binding` 统一改回 `Directive`

`directives.js` 内置指令中加入了 `sd-checked`

这个demo还不完整，`directives.js`中的`mutate` 中还未加相关逻辑，当数组长度改变时（如添加一个todo）并不能反馈到dom上。

```js

update: function (collection) {
    if (this.childSeeds.length) {
        this.childSeeds.forEach(function (child) {
            child.destroy()
        })
        this.childSeeds = []
    }
    watchArray(collection, this.mutate.bind(this))
    var self = this
    collection.forEach(function (item, i) {
        self.childSeeds.push(self.buildItem(item, i, collection))
    })
},
mutate: function (mutation) {
    console.log(mutation)
}
```

---

#### awesome [fcd9544](https://github.com/vuejs/vue/commit/fcd95440901fd33b7e795e383ea8669a763e3297)

加入了内置指令 `sd-class`

---

#### 123 [16a4e05](https://github.com/vuejs/vue/commit/16a4e05a346960bef4b6bc9f5d875c3d4bef2cb5)

#### stop jshint complaining [39760f4](https://github.com/vuejs/vue/commit/39760f49b7ac4a54be17c8f1c1803f581e7eba31) 

没有实质内容

---

#### nested controllers [f1ed54b](https://github.com/vuejs/vue/commit/f1ed54bc84140b083964af12d3fc72c32fc8a085)

作者现在思考在框架中应该包含的内容：

- Template
- Controller
    - Nested Controllers and accessing parent scope
    - Controller inheritance 
- Data
- Data Binding
- Filters
- Computed Properties
- Custom Filter
- Custom Directive

接下来要做的内容：

- complete arrayWatcher

- computed properties (through invoking functions, need to rework the setter triggering mechanism using emitter)
    - （computed properties 基于一个已声明的函数，这里呢就需要在`setter`里面计算依赖啊啥的）

- the data object passed in should become an absolute source of truth, so multiple controllers can bind to the same data (i.e. second seed using it should insert dependency instead of overwriting it)
    - （原始的`data`保留副本，可以让多个实例来用）

- nested properties in scope (kinda hard but try)
    - （nested properties 的确是比较tough的东西）

- parse textNodes



作者做了一下 `nested controllers` 的尝试

```
<div sd-controller="Grandpa">
    <p sd-text="name"></p>

    <div sd-controller="Dad">
        <p><span sd-text="name"></span>, son of <span sd-text="^name"></span></p>

        <div sd-controller="Son">
            <p><span sd-text="name"></span>, son of <span sd-text="^name"></span></p>

            <div sd-controller="Baby">
                <p><span sd-text="name"></span>, son of <span sd-text="^name"></span>, grandson of <span sd-text="^^name"></span> and great-grandson of <span sd-text="$name"></span></p>

            </div>
        </div>
    </div>
</div>
```

作者定义的逻辑是这样的：

> 
> - name -> 自己的scope 
> - ^name -> 父亲的scope
> - ^^name -> 父亲的父亲的scope
> - $name -> root scope 

在这里，先定义了两个正则：

```js
var ancestorKeyRE = /\^/g,
    rootKeyRE = /^\$/
```

这里就是定义个while循环去match例如 `^^name` 这样的东西，找到最终的 `scopeOwner`

```js
if (snr && !isEachKey) {
    scopeOwner = this.parentSeed
} else {
    var ancestors = key.match(ancestorKeyRE),
        root      = key.match(rootKeyRE)
    if (ancestors) {
        key = key.replace(ancestorKeyRE, '')
        var levels = ancestors.length
        while (scopeOwner.parentSeed && levels--) {
            scopeOwner = scopeOwner.parentSeed
        }
    } else if (root) {
        key = key.replace(rootKeyRE, '')
        while (scopeOwner.parentSeed) {
            scopeOwner = scopeOwner.parentSeed
        }
    }
}
```

---
#### solved the setter init invoke [14d0cef](https://github.com/vuejs/vue/commit/14d0cefe6b65e03786eb9811fb39c741d8cd802e)

干掉了统一初始化`scope`的`_dataCopy`属性相关代码, 

```
var binding = {
    value: this.scope[key], // 原先的value是null
    instances: []
}             
```

```
// set initial value
  if (binding.value) {
      directive.update(binding.value) // 初始化时使用这个value
  }
```


---

#### better unbind/destroy [6d81bff](https://github.com/vuejs/vue/commit/6d81bff63a38cd663afffb147d11ca00e38a42be)

调用`unbind(true)`为`destroy`, 调用`unbind()`即为`unbind`

```js
unbind: function (rm) {
    if (this.childSeeds.length) {
        var fn = rm ? 'destroy' : 'unbind'
        this.childSeeds.forEach(function (child) {
            child[fn]()
        })
    }
}
```

改进了`unbind`, 现在是完备的逻辑了

```js
Seed.prototype.unbind = function () {
    var unbind = function (instance) {
        if (instance.unbind) {
            instance.unbind()
        }
    }
    for (var key in this._bindings) {
        this._bindings[key].instances.forEach(unbind)
    }
    this.childSeeds.forEach(function (child) {
        child.unbind()
    })
}
```

作者同时在思考`delegate`的问题。

```js
// TODO probably there's a better way.
// Angular doesn't support delegation either
// any real performance gain?
delegate: function (handler, args) {
    var selector = args[0]
    return function (e) {
        var oe = e.originalEvent,
            target = delegateCheck(oe.target, oe.currentTarget, selector)
        if (target) {
            e.el = target
            e.seed = target.seed
            handler.call(this, e)
        }
    }
}
```

---

#### fix sd-on [8f79a10](https://github.com/vuejs/vue/commit/8f79a10b3684e63b52892caecd3a87d60427aa70)

怒了，把`delegate`整个注掉了。


---

#### computed property progress [3d33458](https://github.com/vuejs/vue/commit/3d33458b601d3c1cf56f0db1e794acc2b161cd2e)

要实现`computed property`就一定需要有`依赖解析`

显然，`依赖解析`最简单的方法是`声明依赖` 

like this:

```
Total: <span sd-text="total < todos"></span> |
Remaining: <span sd-text="remaining"></span> |
Completed: <span sd-text="completed < remaining total"></span>

// computed properties
scope.total = function () {
    return scope.todos.length
}

scope.completed = function () {
    return scope.todos.length - scope.remaining
}
```

上述这种声明方式，声明了`total`依赖`todos`, `completed` 依赖 `remaining` 和 `total`

但是这样对用户来说成本略高。

这个版本中也还未实现调用`total`, `completed`这些声明函数去计算的逻辑。

---

#### event delegation in sd-each [5227248](https://github.com/vuejs/vue/commit/52272487400c22d1829b7a6efbb71483e84a3b7b)

搞回`delegate`, 实现一个*matchSelector 的 `polyfill`

`directives.js` 里面的 `on`,即指令`sd-on` 有以下逻辑

> - bind时，如果是`sd-each`说明是循环，那么需要用`delegate`；定义当前的`selector`及`delegator(父元素)`
> - update时，通过`delegateCheck`找到`delegator`中的`handler`并绑定`eventListener`；



```js
bind: function (handler) {
    if (this.seed.each) {
        this.selector = '[' + this.directiveName + '*="' + this.expression + '"]'
        this.delegator = this.seed.el.parentNode
    }
},
```

但是现在的实现ms是有问题的，因为上面对`this.delegator`赋值的时候，dom还没有渲染完成，这时是没有`parentNode`的

换句话说，`this.delegator`始终为`null`, 绑定的时机不太对。

---


#### break directives into individual files [ca62c95](https://github.com/vuejs/vue/commit/ca62c95a4bbcbc37951eea598c10a88e8e50a549)

拆分了`directives`到不同的文件 

---

#### arrayWatcher [88513c0](https://github.com/vuejs/vue/commit/88513c077d1ca494b075636aa6a6c76257e63f39)


如前所述，对于数组的`mutation`需要做处理。这里定义了一个 `mutationHandlers`，做以下处理

> - push: `self.buildItem` --> `self.container.insertBefore`
> - pop: `pop` 拿出来的元素，当然是直接`$destroy`
> - unshift: `self.buildItem` --> `self.container.insertBefore` --> `self.reorder` 很形象
> - shift: `$destroy` --> `self.reorder` 
> - splice: `$destroy` --> `self.container.insertBefore`
> - sort: `selft.container.insertBefore`

以`push`为例来梳理一下`arrayWatcher`的实现过程：

> - 在scope中执行了push, 进入被watchArray覆写的push方法，执行push
> - 覆写的push方法带着执行mutationHandler， 接着执行mutationHandler[push],传入三个参数：method,args,result
> - 执行 `self.buildItem` , 通过 `cloneNode` 创建一个node
> - 执行 `new Seed()` 创建一个新实例 `spore`, 绑定这个node
> - 把这个`spore` 放到 负责`sd-each` 的 `directive`中的`collection`属性数组中, 这里当然不能再使用push了
> - 执行dom操作 `self.container.insertBefore`

对于 `push` 等不同的函数 `apply` 处理之后，得到的 `result` 显然是不一样的；在`mutationHandlers`里面也对 `result` 进行量身处理。

```js
Object.keys(mutationHandlers).forEach(function (method) {
    arr[method] = function () {
        var result = Array.prototype[method].apply(this, arguments)
        callback({
            method: method,
            args: Array.prototype.slice.call(arguments),
            result: result
        })
    }
})
```

总之这些`arrayWatcher`实现还是挺优雅的。

应该存在的问题是：

- `Array`不止这些方法，是否全搞一次？
- 性能问题

PS:作者的起名还是挺有意思的， `Seed` , `plant`， `spore` ...

---

#### allow overwritting mutationHandlers [343ea29](https://github.com/vuejs/vue/commit/343ea299d0f10d39680b8da6651c3d3ab11ba21c)

`mutationHandlers`可以被覆写。

这是打算弄成api? 按目前的写法显然用户层是不能定义`mutationHandlers`的。而且显然不应该由用户来定义才对啊。。

---

#### better dep parsing [c4f31a5](https://github.com/vuejs/vue/commit/c4f31a5d65ad66fb2bf74612776444648609c0f6)

检测一个`directive`是否有依赖其它`scope data`; 

在这里引入了一个`parseKey` 函数来解析

指令key的写法有几种，比如：

- done:todo.done (sd-class)
- change:toggleTodo (checkbox)
- completed < remaining total （数据的关联计算）

显然现在的声明式依赖只是用`<`符号来声明；于是便有：

```js
var DEPS_RE = /<[^<\|]+/g
```

这里需要避开连续两个`<`符号和声明`filter`的`|`号


如 `completed < remaining total` 这个指令，最终这一段的执行重复利用了`parseKey`, 很优雅：

```js
var depExp = expression.match(DEPS_RE)
    this.deps = depExp
        ? depExp[0].slice(1).trim().split(/\s+/).map(parseKey)
        : null
// depExp === ["< remaining total"]
// depExp[0].slice(1).trim().split(/\s+/)  --> ["remaining", "total"]
// map(parseKey)之后得到this.deps
```


被解析后的结果：

```
this.deps = [{
    arg: null,
    key: 'remaining',
    nesting: false,
    root: false
    }, {
        arg: null,
        key: 'total',
        nesting: false,
        root: false
    }
]
```

接下来是肯定要 do something的：

```
// computed properties
 if (directive.deps) {
        directive.deps.forEach(function (dep) {
            console.log(dep)
        })
    }
```

PS：由用户来声明依赖的方式显然是需要重构的。慢慢看作者的思路。

---

#### computed properties!!! [5acc8a2](https://github.com/vuejs/vue/commit/5acc8a2986c68f9d41baee7878c778d1b47c9f16)

`parseKey`函数中加入了`nesting`和`root`的判断

当依赖（即`directive.deps`）发生变化时调用 `Directive.prototype.refresh` 

```js
// called when a dependency has changed
Directive.prototype.refresh = function () {
    if (this.value) {
        this._update(this.value.call(this.seed.scope))
    }
    if (this.binding.refreshDependents) {
        this.binding.refreshDependents()
    }
}
```

```js
// computed properties
if (directive.deps) {
    directive.deps.forEach(function (dep) {
        var depScope = determinScope(dep, scope),
            depBinding =
                depScope._bindings[dep.key] ||
                depScope._createBinding(dep.key)
        if (!depBinding.dependents) {
            depBinding.dependents = []
            // 每个依赖发生变化时均会触发refresh()
            depBinding.refreshDependents = function () {
                depBinding.dependents.forEach(function (dept) {
                    dept.refresh()
                })
            }
        }
        depBinding.dependents.push(directive)
    })
}
```

对于`watchArray`, 对数组加入了两个辅助方法`replace` 和 `remove`

---

#### todos [dc04a1a](https://github.com/vuejs/vue/commit/dc04a1af6908ce4ad4f89c232c8da0c38a1d92af)

- parse textNodes
- more directives / filters
- nested properties in scope (kinda hard, maybe later)
 
---

#### complete todo example [7d12612](https://github.com/vuejs/vue/commit/7d126127e6c3c234e887c13d35e95aeafdc35de1)

开始战斗力爆表...

照例还是先看`todo`

- parse textNodes?
- method invoke with arguments
- more directives / filters
    - sd-if (终于出来了)
    - sd-route (这是啥)
- nested properties in scope (kinda hard, maybe later)


`todos demo` 大改动，js,css啥的抽出来, 不得不说，css写得还是很有逼格的。。

布局和[todomvc](https://github.com/tastejs/todomvc)差不多。


默认指令添加了 `hide`, `focus`, `value`

`filters`添加了常用的`keyCode`, 加入了`currency` 和 `key` 两个`filter`

---

#### jshint; [19b3926](https://github.com/vuejs/vue/commit/19b39263fce631df030a6259282eb3bafb053ff6)

格式问题

#### todo [f19e6c3](https://github.com/vuejs/vue/commit/f19e6c36f61ab92e62ded53954ecd93436784b98)

- limited set of expressions (e.g. ternary operator)

考虑在指令中支持更多的`expression`如`三目运算符`

#### thoughts [c1c0b3a](https://github.com/vuejs/vue/commit/c1c0b3a036a110cffd351f8a6394b4e00796f8bc)

`todo.md`中的思考:

- getter setter should emit events
- auto dependency extraction for computed properties (evaluate, record triggered getters)
- use descriptor for computed properties


`getter`和`setter`中做`emit`有啥用？暂时没有想到。。

果然在考虑`自动收集依赖`。。

另外`descriptor`（描述符）是个什么鬼。。

今天先这样吧

---

#### use emitters [9a4e5d0](https://github.com/vuejs/vue/commit/9a4e5d035027d8ad570aca6006dd553c715bcde4)

> `getter`和`setter`中做`emit`有啥用？暂时没有想到。。

恩，在这个commit里面得到了解释，就是使用`Emmiter`来做事件监听啦。以这种`事件驱动`的方式来实现。

例如：

```js
    var seed = this
    Object.defineProperty(this.scope, key, {
        get: function () {
            seed.emit('get', key)
            return binding.value
        },
        set: function (value) {
            if (value === binding.value) return
            seed.emit('set', key, value)
        }
    })

    ...

 // update bindings when a property is set
    this.on('set', this._updateBinding.bind(this))
```

---

#### sourceURLs for dev, reverse value [f6d6bba](https://github.com/vuejs/vue/commit/f6d6bba70d8715a49ced83b8c4db06fec4d2cf79)

grunt的工作，uglify & source mapping 

增加对 `!` 取非操作的逻辑。
 
---

#### add simple example & manual refresh of computed properties [67ff344](https://github.com/vuejs/vue/commit/67ff3448109104fe0d5f6910ba5397fe0a5c6a99)

惯例先来看`todo.md` : 

> - sd-width
> - sd-visible
> - sd-style

弄了一个最简的`hello world example`

加了一个`sd-data`进来，有什么用？只能后面再看了
```js
dataSlt = '[' + config.prefix + '-data]'
```

---

#### auto parse dependency for computed properties!!!!! [7a0172d](https://github.com/vuejs/vue/commit/7a0172d60b636c83a9559b6163d3b2d9058759ab)

作者用了这么多感叹号，想必是很爽的。。实现了`依赖的自动计算`

这个commit对`computed properties`做了较大的重构，需要梳理一下：

以下面这个`total`属性为例：

```js
scope.total = {get: function () {
    return scope.todos.length
}}
```

- `total`依赖于`todos`。
- 把依赖计算写到了`get`函数里。（这里规定了这样的格式）

执行了上面的赋值后，必然触发`set`;

```js
 // add event listener to update corresponding binding
// when a property is set
this.on('set', this._updateBinding.bind(this))
```

由这句，去触发`_updateBinding`, 有以下逻辑：

```js
if (type === 'Object') {
    if (value.get) { // computed property
        this._computed.push(binding)
        binding.isComputed = true
        value = value.get
    } else { // normal object
        // TODO watchObject
    }
} 
```

可以看出：

- 如果`value.get`有定义，那么它就是一个`computed property` （isComputed = true）
- `push`到`Seed`的`_computed`属性中

然后开始：

```js
    // update all instances
    binding.instances.forEach(function (instance) {
        instance.update(value)
    })

    // 接着执行
    if (typeof value === 'function' && !this.fn) {
        // 这里就是开始执行scope.todo.length 了
        value = value()
    }
```

那么在这里又会对`Scope.todo`做`get`操作......


```js

Seed.prototype._createBinding = function (key) {
    ...
    Object.defineProperty(this.scope, key, {
        get: function () {
            if (parsingDeps) {
                // 每当get时，就会触发`get`事件
                depsObserver.emit('get', binding)
            }
            seed.emit('get', key)
            return binding.isComputed
                ? binding.value()
                : binding.value
        },
        ...
    })

    return binding
}

// 接着执行 

    this._computed.forEach(this._parseDeps.bind(this))

// 接着执行 

Seed.prototype._parseDeps = function (binding) {
    // 如上, 触发了`get`事件后，这里的dependents就会收集依赖。
    depsObserver.on('get', function (dep) {
        if (!dep.dependents) {
            dep.dependents = []
        }
        dep.dependents.push.apply(dep.dependents, binding.instances)
    })
    binding.value()
    // 收集完依赖，干掉 
    depsObserver.off('get')
}

```

**这个流程就结束了**





> 如果设置了新值，流程如下：

> update --> refresh --> emitChange -->  所有依赖refresh 

```js
// called when a new value is set
Directive.prototype.update = function (value) {
    ...
    if (this.binding.isComputed) {
        this.refresh()
    }
}

// called when a dependency has changed
Directive.prototype.refresh = function () {
    ...
    this.binding.emitChange()
}

 Binding.prototype.emitChange = function () {
     this.dependents.forEach(function (dept) {
         dept.refresh()
     })
 }
```
---

#### fix _dump() [dada181](https://github.com/vuejs/vue/commit/dada181c8581e4af4e1bf16b86744ebdc8d8d35e)

#### no longer need $refresh [faf0557](https://github.com/vuejs/vue/commit/faf055791e8a0f1b36cdeafb41b13f22eb8b9b8b)

#### clean up, trying to fix delegation after array reset [c0a65dd](https://github.com/vuejs/vue/commit/c0a65ddda5f0e6a332b8dbb2bb2552fdef6f5f89)

结合前面的`get`做的改动。
还在和`delegation`做斗争, 使用`each`指令时, 来操作`parentNode` :

> 那么前面一直存在的各种matchSelector就木用了

```js
bind: function () {
    ...
    this.delegator = this.el.parentNode;
}

 unbind: function (rm) {
     ...
        var delegator = this.delegator
        if (!delegator) return
        var handlers = delegator.sdDelegationHandlers
        for (var key in handlers) {
            console.log('remove: ' + key)
            delegator.removeEventListener(handlers[key].event, handlers[key])
        }
        delete delegator.sdDelegationHandlers
    }
```

---

#### separate binding into its own file [832e975](https://github.com/vuejs/vue/commit/832e97588d0a600d2a29ff406ebf1648341e9043)

clean up binding [60e246e](https://github.com/vuejs/vue/commit/60e246eedbd8d0a2be44a5ef2274f506d322b931)

把负责绑定和数组监听的部分拆分到 `binding.js`，部分变量重命名

---

#### sd-if [60a3e46](https://github.com/vuejs/vue/commit/60a3e46fbb8b1f86f29c080f50ef3329e6be40c7)

加入`sd-if`指令。

```js
'if': {
    bind: function () {
        this.parent = this.el.parentNode
        this.ref = this.el.nextSibling
    },
    update: function (value) {
        if (!value) {
            if (this.el.parentNode) {
                this.parent.removeChild(this.el)
            }
        } else {
            if (!this.el.parentNode) {
                // insertBefore时需要考虑原始位置。
                this.parent.insertBefore(this.el, this.ref)
            }
        }
    }
}
```
这里`insertBefore`是需要维护**原来的位置**的，所以要维护一个`参考节点`即`nextSibling`

---

#### sd-style [646b790](https://github.com/vuejs/vue/commit/646b790863110508ec0f074d75d262b4159384fb)

`style`指令。

现在的写法，以`borderColor`为例：

> dom :  sd-style="borderColor"
> scope: borderColor = '#000'

是以`-`做分隔符，比如`borderColor`这个style，写成`border-color`

额，一个初级版本吧。

---

#### remove unused [62a7ebe](https://github.com/vuejs/vue/commit/62a7ebe7db8306f9041880ff52508b23ae0c1d28)

嗯，把各种`matchSelector`干掉了 

---

#### remove redundant dependencies for computed properties [76ee306](https://github.com/vuejs/vue/commit/76ee306bdfa60e19cc3d3981547b64a68af4d2f4)

原先的依赖解析提取出来, 单独作为流程的一步，这样清晰多了。

```js
this._computed.forEach(parseDeps)
this._computed.forEach(injectDeps)
```

同时，原来的依赖解析有`重复依赖`的问题，每次`get`都会去push，要修一下

```js
function parseDeps (binding) {
    depsObserver.on('get', function (dep) {
         if (!dep.dependents) {
             dep.dependents = []
             }
        dep.dependents.push.apply(dep.dependents, binding.instances)
    })
```

放在了binding.dependencies里面：

```js

/*
 *  Auto-extract the dependencies of a computed property
 *  by recording the getters triggered when evaluating it.
 *
 *  However, the first pass will contain duplicate dependencies
 *  for computed properties. It is therefore necessary to do a
 *  second pass in injectDeps()
 */
function parseDeps (binding) {
    binding.dependencies = []
    depsObserver.on('get', function (dep) {
        binding.dependencies.push(dep)
    })
    binding.value.get()
    depsObserver.off('get')
}
```

修掉，原因作者说得很清楚了：

```js

/*
 *  The second pass of dependency extraction.
 *  Only include dependencies that don't have dependencies themselves.
 */
function injectDeps (binding) {
    binding.dependencies.forEach(function (dep) {
        if (!dep.dependencies || !dep.dependencies.length) {
            dep.dependents.push.apply(dep.dependents, binding.instances)
        }
    })
}
```

---











