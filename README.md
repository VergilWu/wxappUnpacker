## 前言

反编译微信小程序，可以拿到小程序源码，但没有project.config.json小程序配置文件。拿到源码后能跑，但登录、授权等涉及到appId的功能是不能用的。另外脚本网上一大堆，原作者github已清，流传的很多版本是不能用的。

## 当前功能如下（分包功能尚未完成！）

- `node wuConfig.js <files...>` 将 app-config.json 中的内容拆分到各个文件对应的 .json 和 app.json , 并通过搜索 app-config.json 所在文件夹下的所有文件尝试将 iconData 还原为 iconPath 。
- `node wuJs.js <files...>` 将 app-service.js (或小游戏中的 game.js ) 拆分成一系列原先独立的 javascript 文件，并使用 Uglify-ES 美化，从而尽可能还原编译前的情况。
- `node wuWxml.js [-m] <files...>` 将编译/混合到 page-frame.html ( 或 app-wxss.js ) 中的 wxml 和 wxs 文件还原为独立的、未编译的文件。如果加上`-m`指令，就会阻止`block`块自动省略，可能帮助解决一些相关过程的 bug 。
- `node wuWxss.js <dirs...>` 通过获取文件夹下的 page-frame.html ( 或 app-wxss.js ) 和其他 html 文件的内容，还原出编译前 wxss 文件的内容。
- `node wuWxapkg.js [-o] [-d] [-s=<Main Dir>] <files...>` 将 wxapkg 文件解包，并将包中上述命令中所提的被编译/混合的文件自动地恢复原状。如果加上`-o`指令，表示仅解包，不做后续操作。如果加上`-d`指令，就会保留编译/混合后所生成的新文件，否则会自动删去这些文件。同时，前面命令中的指令也可直接加在这一命令上。而如果需要解压分包，请先解压主包，然后执行`node wuWxapkg.js [-d] -s=<Main Dir> <subPackages...>`，其中`Main Dir`为主包解压地址。除`-d`与`-s`外，这些指令两两共存的后果是未定义的（当然，是不会有危险的）。

## 所需工具

1. [nodeJs](https://nodejs.org/zh-cn/download/ )
2. 反编译脚本
3. 安卓已root手机或有超级权限的模拟器
4. 文件管理器

## 反编译流程

1. 打开文件管理器，从根目录开始。/data/data/com.tencent.mm/MicroMsg/{User}/appbrand/pkg/，其中有一段`{User}`是代表每个微信账号生成的唯一id，找到小程序包，打包发送至电脑。或者直接手机连接电脑adb

   ```shell
   adb pull /data/data/com.tencent.mm/MicroMsg/{User}/appbrand/pkg
   ```

2. cd到脚本所在目录，安装依赖包`npm install`，如果需要全局安装，就

   ```shell
   npm install esprima -g
   npm install css-tree -g
   npm install cssbeautify -g
   npm install vm2 -g
   npm install uglify-es -g
   npm install js-beautify -g
   npm install escodegen -g
   ```

3. 反编译第一部拿到的微信小程序包。

   ```shell
   解包主包
   
       windows系统使用: ./bingo.bat testpkg/master-xxx.wxapkg
       Linux系统使用: ./bingo.sh testpkg/master-xxx.wxapkg
   
   解包子包
   
       windows系统使用: ./bingo.bat testpkg/sub-1-xxx.wxapkg -s=../master-xxx
       Linux系统使用: ./bingo.sh testpkg/sub-1-xxx.wxapkg -s=../master-xxx
   ```

## 局限（包括但可能不限于以下内容）

- 实现中很多功能基于特定的版本(`wcc-v0.6vv_20180111_fbi`, 且不考虑面向低版本适配)和字符串搜索，所以不能很好的适应各种特殊情况。
- wxml 文件拥有不同于 xml 和 html 文件的字符转义规则，且尚未公开(并非"没有")，因此未能很好的还原相关内容。
- js 文件被压缩后会丢失原始变量名等信息内容无法还原；wxss 文件压缩后的注释也会丢失。
- wxs 文件会将所有的变量如 Math 改为 nv_Math ，这里仅通过字符串替换去除。
- 一些被引用 wxss 文件本身的源文件丢失，因此无法恢复原始目录。
- 有些项目开启了难以复原的`es6转es5`选项，检验本项目结果是否正确时需要关闭项目中的`es6转es5`选项。
- wxml 中一些无法找到相对应 的正向语句的内容无法还原。
- json 中`components`项丢失，仅会标注被其他包引用的自定义组件。

## 另注

所有命令上都可以使用`-f`指令来提高一定的并行度，但输出信息会混乱。

如果发现包内文件“缺失”，请先检查解包时是否出现提示`NOTICE: SubPackages exist in this package.`。如存在，请在寻找好分包后，按上文提示操作。（小程序需要访问特定页面；小游戏需要触发特定函数，然后分包才会被下载。）

> -s 参数可为相对路径或绝对路径, 推荐使用绝对路径, 因为相对路径的起点不是当前目录 而是子包解包后的目录

```
├── testpkg
│   ├── sub-1-xxx.wxapkg #被解析子包
│   └── sub-1-xxx               #相对路径的起点
│       ├── app-service.js
│   ├── master-xxx.wxapkg
│   └── master-xxx             # ../master-xxx 就是这个目录
│       ├── app.json
```