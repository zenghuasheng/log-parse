# Vue 3 + Vite

This template should help get you started developing with Vue 3 in Vite. The template uses Vue 3 `<script setup>` SFCs, check out the [script setup docs](https://v3.vuejs.org/api/sfc-script-setup.html#sfc-script-setup) to learn more.

Learn more about IDE Support for Vue in the [Vue Docs Scaling up Guide](https://vuejs.org/guide/scaling-up/tooling.html#ide-support).

# 初始化 Vite 项目
```shell
npm create vite@latest
```

参考现有代码，写一个 nginx 日志解析应用，首先需要有日志解析功能，写一个接口，接收日志文件和解析表达式，
取文件前 100 行和文件后 100 行，计算MD5, 作为日志存储目录
schema 要写成动态的，每个日志文件都有自己的 schema
先返回前100行的解析结果，让用户确认解析表达式是否正确后再解析全部的

展示功能，支持按日期筛选，按关键字搜索，按 traceID 排序
统计耗时、调用次数
