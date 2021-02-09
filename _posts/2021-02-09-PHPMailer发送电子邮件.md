---
layout: mypost
title: PHPMailer发送电子邮件
categories: [PHP]
---

>   在 PHP 应用开发中，往往需要验证用户邮箱、发送消息通知，而使用 PHP 内置的 mail() 函数，则需要邮件系统的支持。
>
>    如果熟悉 IMAP/SMTP 协议，结合 Socket 功能就可以编写邮件发送程序了，不过开发这样一个程序并不容易。
>
>    好在 PHPMailer 封装的足够强大，使用它可以更加便捷的发送邮件，免去了我们很多额外的麻烦。


## PHPMailer
PHPMailer 是一个封装好的 PHP 邮件发送类，支持发送 HTML 内容的电子邮件，以及可以添加附件发送，并不像 PHP 本身 mail() 函数需要服务器环境支持，您只需要设置邮件服务器以相关信息就能实现邮件发送功能。

项目地址： [https://github.com/PHPMailer/PHPMailer](https://github.com/PHPMailer/PHPMailer){:target="_blank"}

## php扩展支持

PHPMailer 需要 PHP 的 sockets 扩展支持，而登录 QQ 邮箱 SMTP 服务器则必须通过 SSL 加密，故 PHP 还得包含 openssl 的支持。

## 核心文件
![核心代码](019577F9EEDB.png)

## 邮箱准备
所有的主流邮箱都支持 SMTP 协议，但并非所有邮箱都默认开启，您可以在邮箱的设置里面手动开启。
第三方服务在提供了账号和密码之后就可以登录 SMTP 服务器，通过它来控制邮件的中转方式。

以下以qq邮箱为例  
设置邮箱  
![核心代码](FCD83E9338CF.png)
打开SMTP设置
![核心代码](1C739DBDA51B.png)

### 代码实例

````php
// 引入PHPMailer的核心文件
use PHPMailer\PHPMailer\PHPMailer;

//require '../vendor/autoload.php';

//???===文件的引入根据项目具体情况===????

// 实例化PHPMailer核心类
$mail = new PHPMailer();
// 是否启用smtp的debug进行调试 开发环境建议开启 生产环境注释掉即可 默认关闭debug调试模式
$mail->SMTPDebug = 1;
// 使用smtp鉴权方式发送邮件
$mail->isSMTP();
// smtp需要鉴权 这个必须是true
$mail->SMTPAuth = true;
// 链接qq域名邮箱的服务器地址
$mail->Host = 'smtp.qq.com';
// 设置使用ssl加密方式登录鉴权
$mail->SMTPSecure = 'ssl';
// 设置ssl连接smtp服务器的远程服务器端口号
$mail->Port = 465;
// 设置发送的邮件的编码
$mail->CharSet = 'UTF-8';
// 设置发件人昵称 显示在收件人邮件的发件人邮箱地址前的发件人姓名
$mail->FromName = '发件人昵称';
// smtp登录的账号 QQ邮箱即可
$mail->Username = '12345678@qq.com';
// smtp登录的密码 使用生成的授权码
$mail->Password = '**********';
// 设置发件人邮箱地址 同登录账号
$mail->From = '12345678@qq.com';
// 邮件正文是否为html编码 注意此处是一个方法
$mail->isHTML(true);
// 设置收件人邮箱地址
$mail->addAddress('87654321@qq.com');
// 添加多个收件人 则多次调用方法即可
$mail->addAddress('87654321@163.com');
// 添加该邮件的主题
$mail->Subject = '邮件主题';
// 添加邮件正文
$mail->Body = '<h1>Hello World</h1>';
// 为该邮件添加附件
$mail->addAttachment('./example.pdf');
// 发送邮件 返回状态
$status = $mail->send();
````