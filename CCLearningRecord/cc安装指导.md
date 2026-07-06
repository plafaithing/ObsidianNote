## 前期准备工作
### 1. 安装npm
下载链接 https://www.nodejs.com.cn/download.html
安装教程可直接百度搜索，或者参考链接 https://zhuanlan.zhihu.com/p/7314838716
### 2. 安装git
参考链接 https://blog.csdn.net/mukes/article/details/115693833
安装完成后，如果不涉及github相关操作，可不做单独git配置
## 安装CC
1. 键盘win + R ，输入cmd，打开终端
![[Pasted image 20260523205348.png]]
2. 终端命令行输入代码
```bash
npm install -g @anthropic-ai/claude-code
```
等待安装，安装完成后输入 ``` claude --version ``` 有对应版本回显即表示安装成功

3. 重新打开终端，输入claude，会显示下图，表示已经安装成功，此时先关闭终端，进行下一步配置API Key
![[Pasted image 20260523205829.png]]

## API Key配置
### 安装CCSwtich
浏览器搜索```farion1231/cc-switch```
![[Pasted image 20260523210848.png]]
打开该github页面，下拉点击invalid
![[Pasted image 20260523210943.png]]
跳转后页面下拉，点击此处
![[Pasted image 20260523211156.png]]
显示全后，下载windows.msi文件
![[Pasted image 20260523211221.png]]
下载后正常安装，安装地址默认即可
### 配置API Key
安装完成后打开CC Switch，按步骤依次点击
![[Pasted image 20260523211509.png]]
如果已经有智普APIKey，则选择Zhipu GLM。同理，若有DeepSeek，则可选DeepSeek。选择后点击右下角添加
![[Pasted image 20260523211552.png]]

弹出框"配置存在一下问题"不管，直接仍要保存，之后会回到主界面，选择Zhipu GLM，编辑
![[Pasted image 20260523211952.png]]
主要填写API Key，其余一般会默认补全
![[Pasted image 20260523212445.png]]
下方模型映射配置修改
![[Pasted image 20260523212533.png]]

对应能力

|级别|模型|消耗系数|适用场景|
|---|---|---|---|
|Haiku|可选GLM-4.5-Air，我这边选择GLM-5|1×|简单问答、轻量任务|
|Sonnet|GLM-5.1|1×|日常编程、代码补全|
|Opus|推荐GLM-5.1|2-3×|复杂推理、架构设计|
修改完成后保存，点击测试
![[Pasted image 20260523212911.png]]
**参考指标：** 绿色代表延迟在 500ms以内，黄色为500 ~ 1500ms，红色则超过1500ms。如果测速显示为红色，说明当前连接不稳定或请求绕路严重。

## 使用CC
1. 到指定文件夹目录下，右键打开终端，输入cmd，再输入claude
![[Pasted image 20260523213311.png]]
此时cc左侧界面下方显示当前的大模型是什么，也代表我们已经配置成功。
当我们在新的文件夹打开cc时，会出现是否信任该文件夹
![[Pasted image 20260523213451.png]]
选择第一个即可，之后便会出现前面的cc界面
可以正常使用cc了

## skill安装
推荐superpowers先安装，开发较全,在cc界面中执行命令
```
/plugin marketplace add https://github.com/obra/superpowers-marketplace.git
```
如果出现以下问题
```bash
> /plugin install superpowers@superpowers-marketplace

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  Plugins  Discover   Installed   Marketplaces   Errors

  Plugin Details

  superpowers
  Version: 5.1.0

  Core skills library: TDD, debugging, collaboration patterns, and proven techniques

  Will install:
  · Component summary not available for remote plugin

  ‼Make sure you trust a plugin before installing, updating, or using it. Anthropic does not control what MCP servers,
    files, or other software are included in plugins and cannot verify that they will work as intended or that they
   won't change. See each plugin's homepage for more information.

  > Install for you (user scope)
    Install for all collaborators on this repository (project scope)
    Install for you, in this repo only (local scope)
    Back to plugin list这个推荐选哪个
```

推荐选择第一个即可
