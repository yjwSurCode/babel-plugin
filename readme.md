# 一： What is babel?

> ## Babel 是一个 JavaScript 编译器

Babel 是一个工具链，主要用于在当前和旧浏览器或环境中将 ECMAScript 2015+ 代码转换为向后兼容的 JavaScript 版本。

# 二：babel 内部机制

> ## 2.1 `babel 前置知识`

> 2.1.1 `静态分析 VS 动态分析`

静态分析是在不需要执行代码的前提下对代码进行分析的处理过程(还可以对我们的源代码进行优化、压缩等操作)。

动态分析是在代码的运行过程中对代码进行分析和处理。

> 2.1.2 `AST`(它就是一棵'对象树'，用来表示代码的语法结构)

将源码解析成 AST:(<a href="https://astexplorer.net/" target="_blank">在线 AST 转换器</a>
)

```
const a='a'

{
  "type": "Program",
  "start": 0,
  "end": 11,
  "body": [
    {
      "type": "VariableDeclaration",
      "start": 0,
      "end": 11,
      "declarations": [
        {
          "type": "VariableDeclarator",
          "start": 6,
          "end": 11,
          "id": {
            "type": "Identifier",
            "start": 6,
            "end": 7,
            "name": "a"
          },
          "init": {
            "type": "Literal",
            "start": 8,
            "end": 11,
            "value": "a",
            "raw": "'a'"
          }
        }
      ],
      "kind": "const"
    }
  ],
  "sourceType": "module"
}

```

知识点很全!!!!

<a href="https://github.com/babel/babylon/blob/master/ast/spec.md" target="_blank">Babel 的 AST 官方讲解</a>

<a href="https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md#toc-visitors" target="_blank">官方的 handbook</a>

## 2.2 `babel 源码分析`

> 2.2.1 通过 normalizeFile 将传入的文件转化为 AST

```
export function* run(
  config: ResolvedConfig,
  code: string,
  ast?: t.File | t.Program | null,
): Handler<FileResult> {
  // 🌵🌵🌵 1. 将代码转化为 AST 🌵🌵🌵
  const file = yield* normalizeFile(
    config.passes,
    normalizeOptions(config),
    code,
    ast,
  );

  const opts = file.opts;
  try {
  	// 🌵🌵🌵 2. 将 ES6 的 AST 转化为 ES5 的 AST 🌵🌵🌵
    yield* transformFile(file, config.passes);
  } catch (e) {}

  let outputCode, outputMap;
  try {
    if (opts.code !== false) {
    	// 🌵🌵🌵 3. 将 ES5 的 AST 生成 ES5 代码 🌵🌵🌵
      ({ outputCode, outputMap } = generateCode(config.passes, file));
    }
  } catch (e) {}

  return {
    metadata: file.metadata,
    options: opts,
    ast: opts.ast === true ? file.ast : null,
    code: outputCode === undefined ? null : outputCode,
    map: outputMap === undefined ? null : outputMap,
    sourceType: file.ast.program.sourceType,
  };
}
```

> 2.2.2 通过 transformFile 处理 AST 产出新的 AST。

transformFile 方法它执行的时候最终会执行 traverse 方法，traverse 的主要工作是将 ES6 的 AST 转化为 ES5 的 AST,babel 的各种插件也都是基于此实现的，比如 JSX，TS 的转化等。

> 2.2.3 通过 generateCode 将新的 AST 转化为目标代码(Babel 会假设你的目标可能是最旧的浏览器)

```
// babel-generator/src/index.ts

export default function generate(
  ast: t.Node,
  opts?: GeneratorOptions,
  code?: string | { [filename: string]: string },
): any {
  const gen = new Generator(ast, opts, code);
  // 🌵🌵🌵 执行这里 🌵🌵🌵
  return gen.generate();
}

// Generator
generate() {
  // 🌵🌵🌵 执行这里 🌵🌵🌵
  return super.generate(this.ast);
}

// -------------我是快乐的分割线--------------

// Printer
generate(ast) {
  // 🌵🌵🌵 执行这里 🌵🌵🌵
  this.print(ast);
  this._maybeAddAuxComment();

  return this._buf.get();
}
```

# 三： Babel Preset + Babel Plugins 的用法，区别和关联。

```
{
  "presets": [
    ["babel-preset-es2015"],
    [
      "es2015",
      {
        "loose": true,
        "modules": false
      }
    ]
  ],
  "plugins": [["babel-plugin-transform-react-jsx"]]
}

```

## 3.1：`Babel Preset`

可以简单的把 Babel Preset 视为 Babel Plugin 的集合

## 3.1.1`官方预设`

- @babel/preset-env
- @babel/preset-typescript
- @babel/preset-react
- @babel/preset-flow
- 拓展:
- @babel-preset-minify
- @babel-preset-vue
- browserslist

## 3.2：`Babel Plugins`（比 preset 先执行 ）

Babel 的代码转换是通过将插件（或预设）应用到您的配置文件来启用的。

> 原始代码 --> [Babel Plugin] --> 转换后的代码

常用插件：

- @babel-plugin-transform-react-jsx

  介绍：将 JSX 转换为 React 函数调用

- @babel-plugin-dynamic-import-node

  介绍：Babel plugin to transpile import() to a deferred require(), for node (按需加载插件)

- @babel/plugin-syntax-bigint

  介绍：允许解析 BigInt 字面值

- @babel-plugin-syntax-jsx

  介绍：允许解析 jsx

  (了解更多插件：<a href="https://babeljs.io/docs/en/plugins-list" target="_blank">Plugins 列表</a>)

# 四：插件编写 + vite 项目搭建

- 4.1：安装相关依赖包。

  4.1.1：@babel/core，看名字就知道这是 babel 的核心，没他不行，所以首先安装这个包。
  4.1.2：@babel/cli Babel 自带了一个内置的 CLI 命令行工具，可通过命令行编译文件。<BR>
  4.1.3：jest 令人愉快的 JavaScript 测试。

```
npm i babel-core
npm i babel/cli
npm i jest

```

- 4.2：创建 main.ts 文件 自定义插件 (参考：<a href="https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md#toc-visitors" target="_blank">官方的 handbook</a>)

- 4.3 创建 vite-react 项目
  ```
  yarn create vite
  ```
- 4.4 创建 vite.config.js 文件
  <a href="https://vitejs.dev/config/" target="_blank"> 官方地址：https://vitejs.dev/config/</a>
- 4.5 创建.babelrc(babel.config.js)文件调用自定义插件。
