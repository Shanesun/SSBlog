## Mock模块1.0.0

#### 修改记录
修改版本 | 修改人    |     备注    | 日期
:-----: | :------: | :--------: | :-----------:
1.0.0   | Shane    | 初始化版本   |2016.02.25
        |          |             |

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



#### 四、TODO
 * `灵活性 js函数解析`:在模拟Array返回数据时，现在只能模拟一条数据，模拟多条数据会出现 js函数或者key值中带入|n的情况。如果再有更灵活性需求，再考虑加入js函数解析功能。
 * `请求参数 params多级结构`:现版本中只考虑params中有一级参数的情况。如果以后需要使用 params多级情况，再考虑加入 `递归` 排平参数放到url中。
