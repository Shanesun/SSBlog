## Mock模块1.0.1

#### 修改记录
修改版本 | 修改人    |     备注    | 日期
:-----: | :------: | :--------: | :-----------:
1.0.0   | Shane    | 初始化版本   |2016.02.25
1.0.1   | Shane    | 修改TODO    |2016.03.10

#### 概述
为解决后台和移动端并行开发，移动端接入Mock模块，快速调试。Mock模块`支持POST接口`进行Mock处理。`不支持GET`.

#### 一、快速集成
*  check 最新trunk分支
*  在L5中 添加 MockConfig.plist  
*  `mockServer`:对应mock服务器地址。`mockApi`:需要mock的api。  
  ![c1-1](https://raw.githubusercontent.com/Shanesun/SSBlog/master/picture/c1_1.png)

*  `启动应用` 在`设置界面` 开启Mock模式 (默认关闭)。  
![c1-3](https://raw.githubusercontent.com/Shanesun/SSBlog/master/picture/c1_4.png)


开启Mock开关，添加mock服务器地址，添加对应需要mock的api，立即开启mock之旅~

#### 二、trunk修改说明
`WQMockManager`:添加到L1中，用于处理Mock相关逻辑。

`WQRequestObject`:在 `makeDebugRequestURL:param:`中添加 mock 逻辑分支。
```objective-c
  // 启用mock：逻辑开关
  if ([WQMockManager getMockEnabel] &&
    [WQMockManager useMockWithApi:[param objectForKey:@"api"]]) {

      url = [WQMockManager regroupUrlWithParams:param];
  }  // 正常逻辑
  else if (![WQCommons judgeNil:api]) {
    //...
  }
```
`WQSettingVC`:增加 Mock 开关控制。

#### 三、RAP后台支持

   [如何在RAP上配置Mock接口文档](http://doc.anzogame.com/xwiki/bin/view/后台研发/Material/Coordination+Develop+And+RAP+Using/?srid=RboHwRp2)


#### TODO
 * ~~`灵活性 js函数解析`:在模拟Array返回数据时，现在只能模拟一条数据，模拟多条数据会出现 js函数或者key值中带入|n的情况。如果再有更灵活性需求，再考虑加入js函数解析功能。~~  

 > 去掉`jS函数解析`原因：

 > 1 生成含有 function 数据，RAP页面有自己的js处理json中带有的function，生成规范json数据，再展示在页面。所以客户端要达到分析json中带有的function函数，必须在RAP结构网页上剥离js文件用于处理function函数，在iOS JavaScriptCore库中运行，构建正确json数据。在代码中需要较大改动。

 > 2 为了复杂mock数据 编写mock语法本身已经很耗时，背离了mock数据在客户端中 简易、快速 完成接口模型粗步开发的目的。

 * ~~`请求参数 params多级结构`:现版本中只考虑params中有一级参数的情况。如果以后需要使用 params多级情况，再考虑加入 `递归` 排平参数放到url中。~~  

 > 对于 参数params中的多层级，可以将其转化成json传入。
