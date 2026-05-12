//UI删除
1. 测试配置中心删除，前端代码全部删除, 后台相关代码也删除
2. 文曲AI代码全部删除， 后台相关代码也删除
3. 首页小程序管理代码全部删除，后台相关代码删除
4. 百宝箱功能还是保留，去掉小说资源查找和发版前防呆检查

//UI改变
1. 文曲自动化，新增一个入口“上传代码”， 它的交互参考 生成小程序-》向导式创建， 点击确认后，会有结果不停的打印
2. 文曲自动化，编译，发布，批量编译，发布，直接用 miniapp_doc/miniappFactory/simple_package_design.md里面的接口
3. 百宝箱功能去掉，后台代码也去掉


//冗余API
以下接口相关的代码代码，都可以删除
1. AgentAgentAppController
2. AppWeijuMockController
3. CodeSyncController
4. ToolBoxController
5. NovelSearchController
6. SystemConfigController（不确定能不能删除）


//其他的
miniappManagerServer/src/main/java/com/fun/novel/repository/NovelRepository.java
miniappManagerServer/src/main/java/com/fun/novel/facade/NovelAppConfigFacade.java
miniappManagerServer/src/main/java/com/fun/novel/facade/NovelAppQueryFacade.java


//其他需求
1. 任务排队， 应该按照package_id作为唯一去排队。
不同的package_id可以并行

2. 首次上传代码没有appid, 手动填入后，需要往manifest.json中写入； -》todo
   再次上传代码包，如果用户仍然没有填写appid, 又会把代码覆盖。 这个要怎么弄？  -> todo

3. 