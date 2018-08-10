# Fastlane 一键打包
### 一、什么是Fastlane
> The easiest way to build and release mobile apps.  
Fastlane是一套使用Ruby写的自动化工具集，用于iOS和Android的自动化打包、发布等工作，可以节省大量的时间。
![50F602D2-5FDE-4AB5-B865-F67C49247027.png](https://user-gold-cdn.xitu.io/2018/6/11/163ee4ff161b4dee?w=2032&h=706&f=png&s=221556)

![1608265-f63702702cfa790f.png](https://user-gold-cdn.xitu.io/2018/6/11/163ee502e3945308?w=1068&h=1019&f=png&s=74248)

![fastlaneDemo.gif](https://user-gold-cdn.xitu.io/2018/6/11/163ee50522081357?w=1060&h=684&f=gif&s=471747)

[Github开源地址](https://github.com/fastlane/fastlane)
[fastlane官网](https://fastlane.tools))

### 二、用来做什么
iOS一键脚本完成以下步骤：
1 reset_git_repo    移除git中未提交的内容
2 git pull         只拉取最新tag
3 checkout        切换到最新的tag
4 submodule update 更新子模块
5 Pod install        第三方库管理工具
6 Archive、Sign    打包、签名
7 upload fir        上传Fir
8 notification        钉钉机器人 

Android 可以实现一键多渠道打包：
1 reset_git_repo            移除git中未提交的内容
2 git pull                 只拉取最新tag
3 checkout                切换到最新的tag
4 submodule update 更新子模块
5 gradle                    打包
6 add_channels_to_apk     拷贝apk，修改文件名为渠道名，渠道名写入META-INFO目录
参考链接：[Fastlane实战（三）：Fastlane在Android平台的应用](http://www.infoq.com/cn/articles/actual-combat-of-fastlane-part03)

### 三、安装
1) 安装Fastlane
`sudo gem install fastlane`
2) 切换到工程目录初始化
`fastlane init`
3) 运行(lane_name为自定义lane 名字)
`fastlane lane_name `

### 四、使用
1. 构建Fastlane 打包上传 fir
```ruby
# This file contains the fastlane.tools configuration
# You can find the documentation at https://docs.fastlane.tools
#
# For a list of all available actions, check out
#
#     https://docs.fastlane.tools/actions
#
# For a list of all available plugins, check out
#
#     https://docs.fastlane.tools/plugins/available-plugins
#

# Uncomment the line if you want fastlane to automatically update itself
# update_fastlane

fastlane_version "2.96.1"
default_platform(:ios)

platform :ios do
desc "Push a new beta build to fir"
lane :tofir do
# Deletes the Xcode Derived Data
clear_derived_data

# discarding uncommitted changes
reset_git_repo(
force: true,
skip_clean: true,
disregard_gitignore: false
)


# git pull only the tags, no commits
git_pull(only_tags: true)

# get recent tag
recentTagName = last_git_tag

# （fastlane-plugin-git_switch_branch为插件）check out tag
git_switch_branch(branch: recentTagName)

# submodule init and update
git_submodule_update(
init: true,
recursive: true
)

# pod install
cocoapods

scheme = "ProCloser"
ipa_dir  = "./fastlane_build/"+Time.new.strftime('%Y-%m-%d')
ipa_name = recentTagName + "_" + Time.new.strftime('%Y%m%d%H%M')

# gym用来编译 打包 签名
gym(
output_directory: ipa_dir,
output_name: "#{ipa_name}.ipa",
clean: true,
workspace: scheme + ".xcworkspace",
scheme: scheme,
export_xcargs: "-allowProvisioningUpdates",
export_options: {
method: "ad-hoc", # 指定打包方式
teamID: "xxxxxxxxxxx",
thinning: "<none>",
}
)

# （fastlane-plugin-firim为插件）上传ipa到fir.im服务器，在fir.im获取firim_api_token
firim(firim_api_token: "xxxxxxxxxxxxxxx")


desc "Notification 钉钉机器人"
# 钉钉机器人
app_patch   = ipa_dir + "/#{ipa_name}.ipa"
app_version = get_ipa_info_plist_value(ipa: app_patch, key: "CFBundleShortVersionString")
app_build_version = get_ipa_info_plist_value(ipa: app_patch, key: "CFBundleVersion")
app_name    = get_ipa_info_plist_value(ipa: app_patch, key: "CFBundleDisplayName")
app_url     = "xxxxxxxx"
app_icon     = "xxxxxxx"
dingTalk_url = "xxxxxxxxx"

dingMessage = 
{
msgtype: "link", 
link: {
text: "对应git tag为: #{recentTagName}", 
title: "iOS #{app_name} #{app_version} (#{app_build_version}) 测试版本", 
picUrl: "#{app_icon}", 
messageUrl: "#{app_url}"
}
}

uri = URI.parse(dingTalk_url)
https = Net::HTTP.new(uri.host, uri.port)
https.use_ssl = true

request = Net::HTTP::Post.new(uri.request_uri)
request.add_field('Content-Type', 'application/json')
request.body = dingMessage.to_json

response = https.request(request)
puts "--------------- 钉钉机器人已经通知 ✅ ---------------"
puts "Response #{response.code} #{response.message}: #{response.body}"

end
end
```


2. 增加一键远程打包脚本

客户端：
```shell
# 远程到打包机，运行打包机中的shell脚本。在打包机中配置了ssh信任所以不需要输入密码
ssh tiejin@tiejinios.local '/Users/tiejin/closerToFir.sh'
```

服务器：
```shell
# 解锁mac中的keychain，fastlane打包命令中的证书签名需要用到keychain中记录的账户和密码。
security unlock-keychain -p 2018 $KEYCHAIN
# 进入指定目录
cd closer
# 设置环境变量，否则fastlane中涉及到/usr/local/bin/命令部分找不到
export PATH=/usr/local/bin/:$PATH
# 执行fastlane
fastlane tofir
```

3. 和 Jenkins 结合一起使用

3. 自定义Actions
0 什么是Action
Action 是Fastlane中的最小执行单元，Action中封装了一些shell命令。

1 创建自定义Action ，会自动生成xxx.rb文件
`fastlane new_action`

2  编辑 xxx.rb 文件，修改具体逻辑。
```ruby
module Fastlane
module Actions
module SharedValues
REMOVE_TAG_CUSTOM_VALUE = :REMOVE_TAG_CUSTOM_VALUE
end

class RemoveTagAction < Action
def self.run(params)
# 最终要执行的东西，在这里执行

# 1、获取所有输入的参数
# tag 的名称 如 0.1.0
tageName = params[:tag]
# 是否需要删除本地标签
isRemoveLocationTag = params[:isRL]
# 是否需要删除远程标签
isRemoveRemoteTag = params[:isRR]

# 2、定义一个数组，准备往数组里面添加相应的命令
cmds = []

# 删除本地的标签
# git tag -d 标签名称
if isRemoveLocationTag
cmds << "git tag -d #{tageName}"
end

# 删除远程标签
# git push origin :标签名称
if isRemoveRemoteTag
cmds << "git push origin :#{tageName}"
end

# 3、执行数组里面的所有的命令
result = Actions.sh(cmds.join('&'))
UI.message("执行完毕 remove_tag的操作 🚀")
return result

end

#####################################################
# @!group Documentation
#####################################################

def self.description
"输入标签，删除标签"
end

def self.details
# Optional:
# this is your chance to provide a more detailed description of this action
"我们可以使用这个标签来删除git远程的标签\n 使用方式是：\n remove_tag(tag:tagName,isRL:true,isRR:true) \n或者 \nremove_tag(tag:tagName)"
end

# 接收相关的参数
def self.available_options

# Define all options your action supports.

# Below a few examples
[

# 传入tag值的参数描述，不可以忽略<必须输入>，字符串类型，没有默认值
FastlaneCore::ConfigItem.new(key: :tag,
description: "tag 号是多少",
optional:false,# 是不是可以省略
is_string: true, # true: 是不是字符串
),
# 是否删除本地标签
FastlaneCore::ConfigItem.new(key: :isRL,
description: "是否删除本地标签",
optional:true,# 是不是可以省略
is_string: false, # true: 是不是字符串
default_value: true), # 默认值是啥

# 是否删除远程标签
FastlaneCore::ConfigItem.new(key: :isRR,
description: "是否删除远程标签",
optional:true,# 是不是可以省略
is_string: false, # true: 是不是字符串
default_value: true) # 默认值是啥

]
end

def self.output
# Define the shared values you are going to provide
# Example

end

def self.return_value
# If your method provides a return value, you can describe here what it does
nil
end

def self.authors
# So no one will ever forget your contribution to fastlane :) You are awesome btw!
["zhangyan"]
end

# 支持平台
def self.is_supported?(platform)
# you can do things like
#
#  true
#
#  platform == :ios
#
#  [:ios, :mac].include?(platform)
#

platform == :ios
end
end
end
end
```


### 五、深入学习Fastlane
#### 0. Fastlane基本知识
1. Fastlane文件结构

Gemfile 和 Bundle 是什么：
Gemfile文件：gem包管理工具，如：管理fastlane、cocoapods
Appfile文件：从 Apple Developer Portal 获取和项目相关的信息
Fastlane文件：核心文件，主要用于 cli 调用和处理具体的流程
Deliverfile：从 iTunes Connect 获取和项目相关的信息

2. Fastlane的Plugin机制
* Action：对于Fastlane来说Action的收录是非常严格，并且有很强的通用性才能收录其中，即使接收了，整个发布周期也会比较长，而且以后无论是升级还是Bug修复，都依赖Fastlane本身的发版，大大降低了灵活性。 

* Plugin: Plugin就是在Action的基础上做了一层包装，这个包装巧妙的利用了RubyGems这个相当成熟的Ruby库管理系统，所以其可以独立于Fastlane主仓库进行查找，安装，发布和删除。

我们甚至可以简单的认为：Plugin就是RubyGem封装的Action，我们可以像管理RubyGems一样来管理Fastlane的Plugin。

* 使用：
查找
`fastlane search_plugins [query]` 或 [查看所有Plugin](https://link.jianshu.com/?t=https://docs.fastlane.tools/plugins/AvailablePlugins/)

安装
`fastlane add_plugin [plugin_xxx]` 安装完成后 会在fastlane文件夹下生成Pluginfile文件实际就是一个Gemfile文件，里面包含了对于Plugin的引用，格式如下：
```ruby
# Autogenerated by fastlane
#
# Ensure this file is checked in to source control!
# fastlane-plugin-version_from_last_tag 是插件名字
gem 'fastlane-plugin-version_from_last_tag'
```

使用
和Action使用一样。

* 发布Plugin
1 可以作为Gem发布到RubyGems上，这样就可以通过fastlane search_plugins xxx 查找，安装。

2 可以到Github / GitLab上 或者 引用本地路径。
```ruby
gem "fastlane-plugin-version_from_last_tag", git: "https://github.com/jeeftor/fastlane-plugin-version_from_last_tag"
```

发布之前，为了本地调试方便，可以将gem指向本地，在Pluginfile这样声明：
```ruby
gem "fastlane-plugin-version_from_last_tag", path: "../fastlane-plugin-version_from_last_tag"
```


####  1. Fastlane不只是移动端
* 测试方面：
1 UI自动化测试Action: appium  ，以下是appium所有参数
`fastlane action appium`
![401851C4-6DE4-46EC-862E-CEBAB69F1886.png](https://user-gold-cdn.xitu.io/2018/6/11/163ee51067f3424b?w=499&h=621&f=png&s=81812)

2 Android Monkey
下面我们看看Android一般运行Monkey测试的步骤：

Git Pull拉最新的代码
执行./gradlew clean命令清理环境
执行./gradlew assembleTest打包测试版本
执行命令安装最新的测试版本：
`adb shell monkey -p com.wanmeizhensuo.zhensuo -c android.intent.category.LAUNCHER 1`

每次调用命令：
`fastlane do_monkey_test times:3 project:GengmeiAndroid apk_path:./GengmeiAndroid_test.apk ...`
Fastlane文件配置如下：
```ruby
desc "Android monkey test"
lane :do_monkey_test do |options|
times        = options[:times]
project      = options[:project]
apk_path     = options[:apk_path]
package_name = options[:package_name]

hipchat(message: "Start monkey test on #{project}")

git_pull
gradle(task: "clean")
gradle(task: "assembleGmtest")
(1..times.to_i).each do |i|
adb(command: "install -r #{apk_path}")
adb(command: "shell monkey -p #{package_name} -c android.intent.category.LAUNCHER 1")
# 等待30秒，确保闪屏页被Finish后，进入到主页.
sleep(30)
android_monkey(package_name: "#{package_name}", count: '1000', seed: "#{10+i}")
end

hipchat(message: "Execute monkey test on project #{project} successfully")
end
```
自定义的Action  [android_monkey](https://github.com/GengmeiRD/Fastfiles/blob/master/fastlane/actions/android_monkey.rb) 封装了 Android Monkey adb

* 理论上所有频繁使用shell操作的，都可以利用fastlane整合，自动化处理。
Fastlane还有以下这些集成好的Actions：
**ssh**
**scp**
**cli-table**
**cli args parser**

更多Actions和Plugins查阅官网
[Actions - fastlane docs](https://docs.fastlane.tools/actions/)
[Plugins - fastlane docs](https://docs.fastlane.tools/plugins/available-plugins/)

#### 2. 高级用法
1. 前置方法
多个lane是可能包含很多共同的方法，fastlane给出的方案：
```ruby
before_all do |lane, options|
#git_pull
#cocoapods
end
```

2. 后置方法
所有lane执行完时，可能都会有通知逻辑：
```ruby
after_all do |lane,options|
slack(message: "fastlane was successful", success: true)
end
```

3. error处理，每个lane执行时可能都会遇到错误，这个时候都应该有消息通知到大家。error就是一个全局错误处理方法：
```ruby
error do |lane, exception|
slack(message: exception.message, success: false)
end
```

4. 引用机制
远程引用 `import_from_git` [import_from_git - fastlane docs](https://docs.fastlane.tools/actions/import_from_git/)每次执行fastlane的命令时，首先会从git上将需要的文件clone这个项目到本地的临时文件夹中，然后再执行相关命令。如果某个项目中，你不想使用远端上的某个lane，可在项目中的Fastfile中复写这个lane即可。

5. 更多高级用法查看文档 [Advanced - fastlane docs](https://docs.fastlane.tools/advanced/)

* 参考资料
官方文档：
https://docs.fastlane.tools
https://github.com/fastlane/fastlane

[0. Fastlane - iOS 和 Android 自动化构建工具](https://zhuanlan.zhihu.com/p/26544793)
[Fastlane实战（一）：移动开发自动化之道 - 简书](https://www.jianshu.com/p/1aebb0854c78)
[Fastlane实战（二）：Action和Plugin机制 - 简书](https://www.jianshu.com/p/0520192c9bd7)
[Fastlane实战（三）：Android平台上的应用 - 简书](https://www.jianshu.com/p/e68c2ea524c0)

### Tips
1. `fastlane install`命令在bundle update 卡住，gem ruby 源有问题，这里建议添加国内源。

2. ruby镜像
`https://gems.ruby-china.org `
`https://ruby.taobao.org/` 

替换命令：
```ruby
$ gem sources --add https://gems.ruby-china.org/ --remove https://rubygems.org/
$ gem sources -l
*** CURRENT SOURCES ***

https://gems.ruby-china.org
# 请确保只有 gems.ruby-china.org
$ gem install rails
```

3. 远程SSH到打包机执行打包 
报错：code sign 问题导致 unknown error -1=ffffffffffffffff
[iOS远程自动打包问题 - 简书](https://www.jianshu.com/p/b03e59560d31)

#博客 #iOS/CI#

#博客 #iOS/CI#
