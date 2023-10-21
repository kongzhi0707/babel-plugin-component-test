### 组件库实现按需加载

#### 1）搭建一个简单的组件库

可以从 ElementUI 里面复制两个组件为: Alert 和 Tag，我们将组件库命名为 vue-component, 目录结构如下：

```
|--- vue-component
| |--- packages
| | |--- alert
| | | |--- src
| | | | |--- main.vue
| | | |--- index.js
| | |--- tag
| | | |--- src
| | | | |--- tag.vue
| | | |--- index.js
| |--- theme
| | |--- alert.css
| | |--- index.css
| | |--- tag.css
| |--- index.js
| |--- package.json
| |--- readme.md
```

packages 文件夹目录下有两个组件，分别为 alert 和 tag，每个组件都是单独的一个文件夹。基本的机构是一个 js 文件和一个 vue 文件。组件支持使用 Vue.component 注册，也支持插件 Vue.use 方式注册。下面我们分别看看他们的源码是如下：

packages/alert/src/main.vue 代码如下

```
<template>
  <div></div>
  ... 更多代码
</template>

<script type="text/babel">
  // 更多代码
</script>
```

packages/alert/index.js 代码如下：

```
import Alert from './src/main';

Alert.install = function(Vue) {
  Vue.component(Alert.name, Alert);
};

export {
  Alert
}

export default Alert;
```

如上给组件添加了一个 install 方法，后面就可以通过使用 Vue.use(Alert) 来注册。

packages/tag/src/main.vue 代码如下

```
<template>
  <div></div>
  ... 更多代码
</template>

<script type="text/babel">
  // 更多代码
</script>
```

packages/tag/index.js 代码如下：

```
import Tag from './src/tag';

Tag.install = function(Vue) {
  Vue.component(Tag.name, Tag);
};

export {
  Tag
}

export default Tag;
```

组件的样式是放在 theme 文件目录下的，也是每个组件一个样式文件，theme/index.css 包含了所有组件的样式。

最外层还有一个 index.js 文件，该文件是用来作为入口文件导出所有组件的，代码如下:

vue-component/index.js 代码如下：

```
import Alert from './packages/alert/index.js';
import Tag from './packages/tag/index.js';

const components = [
  Alert,
  Tag
];

const install = function (Vue) {
  components.forEach(component => {
    Vue.component(component.name, component);
  })
}

if (typeof window !== 'undefined' && window.Vue) {
  install(window.Vue);
}

export default {
  install,
  Alert,
  Tag,
}
```

如上代码，引入所有的组件，然后提供一个 install 方法，遍历所有的组件，依次使用 Vue.component 方法注册，然后判断是否存在全局的 Vue 对象，如果存在的话，看你是 CDN 方式引入的，那么自动进行注册，最后导出 install 方法 和 所有组件。

#### 使用 Vue 组件

直接引入所有的组件

```
import VueComponent from 'vue-component';
import 'vue-component/theme/index.css';
Vue.use(VueComponent);
```

也可以单独注册某个组件

```
import VueComponent from 'vue-component';
import 'vue-component/theme/alert.css';
Vue.use(VueComponent.Alert);
```

那我们为什么不能通过 import { Alert } from 'vue-component' 来引入组件呢？因为会报错的。现在我们可以来测试下。

#### 测试 Vue 组件的使用方式

现在我们通过 Vue CLI 搭建一个测试项目。运行 npm link vue-component 来链接到组件库，然后使用前面的方式组册组件库或某个组件。

```
vue create test-componet
```

如上创建一个 vue2 的项目。

现在 上面的 vue-component 根目录下 执行 npm link , 把该组件链接到全局。 然后在 test-component 组件的根目录下 执行 npm link vue-component 链接到该组件。

然后在我们的 test-component/src/main.js 引入文件代码

```
import VueComponent from 'vue-component';
import 'vue-component/theme/alert.css';
Vue.use(VueComponent.Alert);
console.log('---VueComponent---', VueComponent);
```

或

```
import VueComponent from 'vue-component';
import 'vue-component/theme/index.css';
Vue.use(VueComponent);
console.log('---VueComponent---', VueComponent);
```

最后我们执行 npm run build 进行打包，发现无论是注册所有的组件，还是只注册 Alert 组件，最后打包的 app.xx.js 中都存在 Tag 组件的内容。如下

<img src="https://raw.githubusercontent.com/kongzhi0707/babel-plugin-component-test
/master/images/1.png" /> <br />

这不是我们想要的按需加载组件的，我们想要的是，每个组件都可以单独为一个组件。 然后我们引入了一个 Alert 组件，不希望 tag 组件的代码也存在。
比如如下这种引入 Alert 组件。

```
import Alert from 'vue/component/packages/alert'
import 'vue-component/theme/alert.css';
Vue.use(Alert);
```

或 最好的引入方式如下就可以引入进去了：

```
import { Alert } from 'vue-component';
```

#### 通过 babel 插件

使用 babel 插件是大多数组件库实现按需加载引入的方式，比如 ElementUI 使用的是 babel-plugin-component；
我们的目标是实现 直接使用 import { Alert } from 'vue-component' 方式来引入 Alert 组件，也不需要手动引入样式。如果要实现这种功能，我们应该要怎么做呢？

我们需要像下面这样这样引入:

```
import { Alert } from 'vue-component';
```

实际按需引用需要像下面这样：

```
import Alert from 'vue-component/packages/alert';
```

因此我们只需要把第一种方式转换成第二种就可以了，而通过 babel 插件来转换就可以了。
因此我们只需要在根目录新建一个 babel-plugin-component.js 文件，作为我们的 babel 插件文件。然后在我们的 babel.config.js 文件引入我们的这个插件文件就可以了。引入方式如下：

```
module.exports = {
  // ....
  plugins: ['./babel-plugin-component.js']
}
```

我们先来看下 import { Alert } from 'vue-component' 对应的 AST 树是什么样的？ <a href="https://astexplorer.net/">在线转换 AST 树</a>

```
{
  "type": "Program",
  "start": 0,
  "end": 37,
  "body": [
    {
      "type": "ImportDeclaration",
      "start": 0,
      "end": 37,
      "specifiers": [
        {
          "type": "ImportSpecifier",
          "start": 9,
          "end": 14,
          "imported": {
            "type": "Identifier",
            "start": 9,
            "end": 14,
            "name": "Alert"
          },
          "local": {
            "type": "Identifier",
            "start": 9,
            "end": 14,
            "name": "Alert"
          }
        }
      ],
      "source": {
        "type": "Literal",
        "start": 22,
        "end": 37,
        "value": "vue-component",
        "raw": "'vue-component'"
      }
    }
  ],
  "sourceType": "module"
}
```

整体是一个 ImportDeclaration，通过 source.value 可以判断导入的来源是 vue-component，specifiers 数组里可以找到导入的变量，每个变量是一个
ImportDeclaration，里面有两个对象：ImportDeclaration.imported 和 ImportDeclaration.local。这两者有什么区别呢，区别就是是否使用了别名导入。比如 我们现在 把 引入方式改成如下：

```
import { Alert as kz } from 'vue-component'
```

那么他们的 AST 抽象语法树变成如下了：

```
{
  "type": "Program",
  "start": 0,
  "end": 43,
  "body": [
    {
      "type": "ImportDeclaration",
      "start": 0,
      "end": 43,
      "specifiers": [
        {
          "type": "ImportSpecifier",
          "start": 9,
          "end": 20,
          "imported": {
            "type": "Identifier",
            "start": 9,
            "end": 14,
            "name": "Alert"
          },
          "local": {
            "type": "Identifier",
            "start": 18,
            "end": 20,
            "name": "kz"
          }
        }
      ],
      "source": {
        "type": "Literal",
        "start": 28,
        "end": 43,
        "value": "vue-component",
        "raw": "'vue-component'"
      }
    }
  ],
  "sourceType": "module"
}
```

别名的情况暂时不考虑。我们现在的问题是如何把 import { Alert } from 'vue-component' 转换成 import Alert from 'vue-component/packages/alert' 的 AST 的结构树。

import Alert from 'vue-component/packages/alert' 的 AST 语法树被转换成如下数据：

```
{
  "type": "Program",
  "start": 0,
  "end": 48,
  "body": [
    {
      "type": "ImportDeclaration",
      "start": 0,
      "end": 48,
      "specifiers": [
        {
          "type": "ImportDefaultSpecifier",
          "start": 7,
          "end": 12,
          "local": {
            "type": "Identifier",
            "start": 7,
            "end": 12,
            "name": "Alert"
          }
        }
      ],
      "source": {
        "type": "Literal",
        "start": 18,
        "end": 48,
        "value": "vue-component/packages/alert",
        "raw": "'vue-component/packages/alert'"
      }
    }
  ],
  "sourceType": "module"
}
```

对比两者的 AST 树结构，我们只需要遍历 specifiers 数组创建新的 ImportDefaultSpecifier 节点，然后替换掉原来的节点即可；因此 babel-plugin-component.js 代码如下：

```
// babel-plugin-component.js
module.exports = ({
  types
}) => {
  return {
    visitor: {
      ImportDeclaration(path) {
        const { node } = path;
        const { value } = node.source;
        if (value === 'vue-component') {
          // 找出引入的组件名称列表
          let specifiersList = [];
          node.specifiers.forEach(spec => {
            if (types.isImportSpecifier(spec)) {
              specifiersList.push(spec.imported.name);
            }
          });
          // 给每个组件创建一条导入语句
          const ImportDeclarationList = specifiersList.map((name) => {
            // 文件夹的名称首字母为小写
            let lowerCaseName = name.toLowerCase();
            // 构造 ImportDeclaration 节点
            return types.importDeclaration([
              types.importDefaultSpecifier(types.identifier(name))
            ], types.stringLiteral('vue-component/packages/' + lowerCaseName))
          })
          // 用于多节点替换单节点
          path.replaceWithMultiple(ImportDeclarationList);
        }
      }
    }
  }
}

```

现在我们对 test-component 代码改成如下， 然后重新打包测试下；

test-component/src/main.js 代码改成如下：

```
import Vue from 'vue'
import App from './App.vue'

import { Alert } from 'vue-component';
import 'vue-component/theme/index.css';
Vue.use(Alert);

Vue.config.productionTip = false

new Vue({
  render: h => h(App),
}).$mount('#app')

```

现在我们对 test-component 项目重新使用 npm run build 打包下，我们继续搜索下 dist/js/app.js 里面发现 Tag 组件内容已经没有了，Alert 组件代码还是有的。

如上是使用 babel 插件 简单实现按需加载文件，我们还有别名也需要处理下， 样式的引入我们上面是全局引入的。后面我们可以看下 <a href="https://github.com/ElementUI/babel-plugin-component">babel-plugin-component </a> 的源码是如何做的。Vant 和 antd 也是采用这种方式做的。他们使用的插件是 <a href="https://github.com/umijs/babel-plugin-import">babel-plugin-import</a>
