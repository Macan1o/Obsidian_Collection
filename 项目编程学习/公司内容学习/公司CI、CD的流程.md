
---

开发提交代码到 Gitee；
方案 A：手动打开 Jenkins 任务，勾选：开启混淆、开启 A 面、不注入垃圾代码；
Jenkins 流水线启动，params.isProguard=true，params.isBuildA=true；
执行 shell 调用 gradle，把这三个布尔值通过 -P传给 gradle；
gradle 读取参数，根据开关：开启混淆、编译 A 面代码，跳过垃圾代码注入；
编译打包输出 apk；
Jenkins 保存 apk 产物，可以下载。