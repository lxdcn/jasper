---
layout: post
cover: assets/images/cover.jpg
subclass: post
title: "用OSX的say命令让Nagios报警叫出来"
navigation: true
logo: assets/images/host.png
date: 2014-09-04 12:34:56
categories: lxd
tags: tech misc shell nodejs nagios chs
comments: true
---

Mac OSX系统有个有趣的命令`say`，传入一段文本就可以让系统读出来。比如你可以打开OSX的Terminal.app，然后输入`say hello OSX`，听听会不会响。

因为机缘巧合，我们项目的CI服务器装在一台Mac Pro上了，在写脚本监控CI Server进程或者做CI monitor的报警的时候，就可以用这个`say`命令把一些信息读出来，再在Mac上插个大喇叭，这样整个项目组都能听得见了。

比如：`say Go server went down, no dzuo no die, ha ha ha ha ha`

当然这样还不够完美，`say`命令只能在Mac上写脚本的时候可以用，于是我们可以写一个HTTP Server，把`say`命令包装一下，放在Mac Pro上跑起来，代码可以是这个样子的：

{% highlight javascript %}
var http = require('http');
var exec = require('child_process').exec; 

http.createServer(function (req, res) {
	var body = "";
	req.on('data', function (chunk) {
		body += chunk;
	});
	req.on('end', function () {
		res.end();
		exec('say ' + body);
	});
}).listen(7788);

console.log('http say now listen on 7788');
{% endhighlight %}

然后我们在别的机器上，POST数据到这台Mac，这台Mac就可以念出来了，比如这样：`curl -X POST -d "No dzuo no die" <MAC_IP_addr>:7788`


Nagios的报警缺省是发邮件的，很不方便，那么就可以tweak一下Nagios发邮件的配置，让它把消息文本POST到Mac上去念出来，config的配置修改是这样的：
{% highlight python %}
# From
define command{
        command_name    notify-service-by-email
        command_line    /usr/bin/printf "%b" "***** Icinga *****\n\nNotification Type: $NOTIFICATIONTYPE$\n\nService: $SERVICEDESC$\nHost: $HOSTALIAS$\nAddress: $HOSTADDRESS$\nState: $SERVICESTATE$\n\nDate/Time: $LONGDATETIME$\n\nAdditional Info:\n\n$SERVICEOUTPUT$\n" | /usr/bin/mail -s "** $NOTIFICATIONTYPE$ Service Alert: $HOSTALIAS$/$SERVICEDESC$ is $SERVICESTATE$ **" $CONTACTEMAIL$
}

# To
define command{
        command_name    notify-service-by-http-say
        command_line    /usr/bin/printf "%b" "$NOTIFICATIONTYPE$, $SERVICEDESC$ on Host $HOSTALIAS$. $SERVICEOUTPUT$" | /usr/bin/curl -X POST -d @- 10.18.7.153:7788
}
{% endhighlight %}

curl的`-d`开关的值`@-`是让curl从标准输入读数据。

一点奇技淫巧，自娱自乐一下 :)
