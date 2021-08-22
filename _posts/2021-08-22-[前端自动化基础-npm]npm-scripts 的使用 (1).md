---
layout: post
title:  "【前端自动化基础-npm】npm-scripts 的使用 (1)"
date:   2021-08-22 23:23:15 +0800
categories: npm npm-scripts npm-cli
---

## 场景

1. 同一个项目，团队人有使用npm安装有人使用yarn（基于两者的安装策略，可能会造成install的安装的版本有差异），安装的版本有差异，会导致使用方式有些许差异，无法保障一致性；

2. 项目pull后，可能会导致本地的node_modules和package.json有差异，直接运行会报错，但仍无法知道增量的安装package是哪些；

3. 由于项目的多人开发场景，每个人在解决业务问题时会根据自己喜好安装npm包，但可能存在已安装的包已经包含预安装包的功能，但是安装的人并不知晓；

   ...

## 引出问题

以上的这几个场景，其产生的问题是很常见的，产生的问题的原因显而易见，在于“人的变动性”，即无流程约束：因此自动化的工具意义在于，排除人的动态因素，流程化的处理任务，约束行为，避免问题。

上面的场景只是项目中问题的冰山一角，今日先从以上的问题下手，慢慢构建一个xx-scripts包，用以流程化处理项目流程。

基于[npm scripts](https://docs.npmjs.com/cli/v7/using-npm/scripts)，来开发我们需要的工具。

## 流程设计

npm运行指令时，会经历一些固定的生命周期，先看下经常使用的npm install/npm publish指令执行时会触发哪些。

<img src="https://github.com/qianghe/blogs/blob/main/imgs/npm-install-lifecycle.png?raw=true" alt="MediaQueryList Support" style="zoom:40%;" />

<img src="https://github.com/qianghe/blogs/blob/main/imgs/npm-publish-lifecycle.png?raw=true" alt="MediaQueryList Support" style="zoom:40%;" />

那么，基于指令必经的生命周期，可以指定在某个生命周期执行一些任务，如下：

<img src="https://github.com/qianghe/blogs/blob/main/imgs/npm-script-logic.png?raw=true" alt="MediaQueryList Support" style="zoom:40%;" />

## 代码实现

1. package.json中指定bin配置，用户安装后，可执行执行custom-scripts:

   ```javascript
   "bin": {
   	"custom-scripts": "./cli.js"
   },
   ```

2. cli.js中根据custom-scripts提供的command及options来分发任务：

   ```javascript
   #!/usr/bin/env node
   
   const exitsNodeModule = require('./helpers/exitsNodeModule')
   if (!exitsNodeModule()) {
   	process.exit(0)
   }
   
   const program = require('commander')
   const chalk = require('chalk')
   
   // base info
   program
   	.version(`${require('./package').version}`)
   	.usage('<command> [options]')
   
   // pre-check command
   program
   	.command('pre-check')
   	.description('help check whether exits packages are not installed')
   	.option('-s, --skip', 'if not exits, return true')
   	.option('-n, --next <exec>', 'if pre-check is ok, support next exec command')
   	.action((options) => {
   		console.log('options::', options)
   		require('./npm/pre-check')(options)
   	})
   
   // --help tooltip
   program.on('--help', () => {
   	console.log()
   	console.log(`  Run ${chalk.cyan(`hq-scripts <command> --help`)} for detailed usage of given command.`)
   	console.log()
   })
   
   program.parse(process.argv)
   ```

3. pre-check中处理逻辑：

   ```javascript
   // ...
   module.exports = async ({
   	skip = false,
   	next: nextExecCommand
   }) => {
     // 加载异步模块（esm）
     await loadGlobalTools()
     // 检测未安装情况
     const checkResult = await checkStatus()
     // ...异常处理
     const [[ unInstallDepPacks, unInstallDevDepPacks ], packageJson] = checkResult
     if (unInstallDepPacks.length > 0 || unInstallDevDepPacks > 0) {
       const unInstallPacks = {
         dep: unInstallDepPacks || [],
         devDep: unInstallDevDepPacks || []
       }
       const needInstall = await invokeInstallQuestion(unInstallPacks)
       
       if (needInstall) {
         await installPackages(unInstallDepPacks.concat(unInstallDevDepPacks), packageJson)
       }
     }
   
     if (nextExecCommand) {
   		shellHelper.exec(nextExecCommand)
   	} else if (skip) {
   		return true
   	} else {
   		process.exit(0)	
   	}
   }
   ```

4. package.json中指定custom-scripts的执行

   ```javascript
   scripts: {
     "start": "custom-scripts pre-check --next='npm run dev'",
     "dev": "xxx"
     ...
   }
   ```

5. 运行指令，结果如下：

   <img src="https://tva1.sinaimg.cn/large/008i3skNly1gtpz5l728kj60pw09iq4102.jpg" alt="截屏2021-08-22 下午11.13.35" style="zoom:67%;" />

## 总结

以上，根据提出的场景问题，我们了解了通过npm scripts生命周期可以帮助我们构建工具解决问题；并通过一个npm run xxx的demo展示了，如何来解决相应的问题。之后继续就场景，进行工具的强化。

代码库：[custom-scripts](https://github.com/qianghe/hq-scripts)

