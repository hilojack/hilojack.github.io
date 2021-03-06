---
title: Typescript Install
date: 2019-10-04
private:
---
# Typescript Install
    npm install -g typescript

# hello.ts

    function sayHello(person: string) {
        return 'Hello, ' + person;
    }

    let user = 'Tom';
    console.log(sayHello(user));

run:

    tsc hello.ts

这时候会生成一个编译好的文件 hello.js：

    function sayHello(person) {
        return 'Hello, ' + person;
    }
    var user = 'Tom';
    console.log(sayHello(user));

如果类型错误，编译会提示，但是也会成功编译出文件。
1. 如果想类型错误时，不生成js 文件, 可以在 tsconfig.json 中配置 noEmitOnError 即可

## 编译为es6
如果想编译为es6, 我们生成编译配置 tsconfig.json

    $ tsc --init; #默认是es5+commonjs 的tsconfig
    $ vi tsconfig.json ;#手动配置
        "target":"es2016"
        "module":"es2016"

默认tsconfig.json 是严格的:`strict:true`

### lib 支持es2020/dom
    $ vi tsconfig.json
    "compilerOptions": {
        "lib": [ "es2020", "dom" ]
    }

### 支持@根目录
      "compilerOptions": {
        "baseUrl": ".",
        "paths": { 
            "@/*": ["./src/*"],
         }
      },
    
### jest path mapping: 
https://kulshekhar.github.io/ts-jest/user/config/
jest 默认使用jest.config.ts

    // jest.config.js
    const { pathsToModuleNameMapper } = require('ts-jest/utils');

    // In the following statement, replace `./tsconfig` with the path to your `tsconfig` file
    const { compilerOptions } = require('./tsconfig');

    module.exports = {
        //moduleNameMapper:{ '^@/(.*)$': '<rootDir>/src/$1', },
        moduleNameMapper: pathsToModuleNameMapper(compilerOptions.paths, { prefix: '<rootDir>/' }  )
    };

### 编译单文件

    # 编译ts
    tsc hello.ts; # 编译单文件不使用tsconfig.json, 而是(`strict:false`)

### 指定config

    # 手动指定配置，编译当前目录下的所有.ts
    $ tsc --p tsconfig.json;

编译为commonjs: see ts-module

    tsc --module commonjs a.ts

### exclude 无关文件
ts 编译时，通过tsconfig.json 指定无关文件

  "exclude": [
    "node_modules",
    "build",
    "scripts",
    "acceptance-tests",
    "webpack",
    "jest",
    "src/setupTests.ts",
    "tslint:latest",
    "tslint-config-prettier"
  ]
## exclude
    "exclude": [
        "node_modules",
        "build",
        "dist",
        "scripts",
        "acceptance-tests",
        "webpack",
        "jest",
        "src/setupTests.ts",
        "tslint:latest",
        "tslint-config-prettier"
    ]


## module 的编译与引入
ts 的模块其实就是es6 module. 比如 a.ts

    export default {
        name: 'a'
    }
    export const value = 'ahui';

b.ts

    import a as name from "./a.js";
    import * as a from "./a.js";

编译得到commonjs规范的 js 文件

    tsc --module commonjs a.ts
    exports.mainValidator = ZipCodeValidator;
    exports.__esModule = true;
    exports["default"] = {
        name: 'a'
    };

其他规范:

    tsc --module es2015 a.ts
    tsc --module amd a.ts
    tsc --module umd a.ts
    tsc --module system a.ts

## 引入json
tsconfig.json:

    "compilerOptions": {
        "resolveJsonModule": true,



## 代码补全d.ts
输入关键字后，vscode 会自动基于`ts`补全。但是对于纯js 文件、第三方库js，由于没有强类型，很难做到补全。我们可以手写`.d.ts`. 

`ts` 也可以生成`.d.ts`:

    tsc hello.ts -d

# 参考
1. ts指南：https://github.com/xcatliu/typescript-tutorial/blob/master/introduction/hello-typescript.md

