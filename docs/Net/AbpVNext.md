# AbpVnext 框架学习和改造

> 本文将以扩展和改造 abpvnext 框架为案例和最终目标学习 abp 框架的使用，底层基于 abp 的 DDD 结构，完善租户和 RBCA，前后端分离

### 一、创建新项目

- AbpCli 脚手架工具的使用

  - 安装

    `dotnet tool install -g Volo.Abp.Cli`

  - 更新

    `dotnet tool update -g Volo.Abp.Cli`

  - 新建项目

    > abp new <解决方案名称> [options]

    - -t 选择模板 app,module,console
    - -u 选择 ui
      - mvc
        - --tiered
          > 创建分层解决方案,Web 和 Http Api 层在物理上是分开的.如果未指定会创建一个分层的解决方案,此解决方案没有那么复杂,适合大多数场景.
      - angular
      - blazor
      - none
        - --separate-identity-server
          > 上面三者共用，将 Identity Server 应用程序与 API host 应用程序分开. 如果未指定,则服务器端将只有一个端点.
    - -m 选择移动应用框架
      - none 不包含移动应用程序
      - react-native React Native
    - -d 选择数据库驱动程序，默认 ef
      - ef EF
      - mongodb Mongo
    - -o 指定输出目录
    - -v 指定版本
    - --preview 使用最新预览版本
    - -csf 项目是否创建在新文件夹
    - -cs 指定数据库连接字符串
    - -dbms 指定数据库
      - SqlServer
      - MySQL
      - SQLite
      - Oracle
      - Oracle-Devart
      - PostgreSQL

  - 更新 Abp 版本

    `Abp update`

  - 实操，创建一个名为 abpfree 的应用,不要 ui，不要移动端 放在 d;\myprograms 创建新目录：
    ` abp new abpfree -t app -u none --separate-identity-server -m none -o d:\myprograms -csf`

- 官网创建项目模板
