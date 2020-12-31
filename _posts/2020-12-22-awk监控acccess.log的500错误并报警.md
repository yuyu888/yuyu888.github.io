---
layout: mypost
title: awk监控access.log里的500错误并报警
categories: [AWK]
---

通过awk监控access.log 里的500错误并通过叮叮接口报警

monitor.sh

````sh
#!/bin/bash
sendMessage(){
    curl 'https://oapi.dingtalk.com/robot/send?access_token=xxxxxx' \
   -H 'Content-Type: application/json' \
   -d '{"msgtype":"link","link":{"text":"请求出现了500错误，请关注","title":"nginx_status_500","picUrl":"","messageUrl":"https://es-xxx.kibana.tencentelasticsearch.com:5601/goto/xxxxxxx"}}'
}
export -f sendMessage
num=`cat cursor.log`
currentNum=`wc -l access.log|awk '{print $1}'`
echo $currentNum>cursor.log
if [ $currentNum -lt $num ];then
echo $currentNum;
echo $num;
echo "新的一天开始了.....";
exit;
fi
awk  -v prenum="$num" 'NR>prenum{if($12=="-"&&$13==500){print NR;print $0;system("sendMessage");exit;}}' access.log
````

=======================acccess.log 样本============================
````
120.133.42.133 - www.xxxx.cn - [14/Oct/2020:15:14:59 +0800] "GET /admin/xxxx/tools/generate-cdkey?length=330&strlen=10 HTTP/1.1" - - - 500 24 "http://wiki.xxxx.net/pages/viewpage.action?pageId=45258367&appid=wiki&rsptoken=xxxx%2BUa1O6yDayL7ORdtPhdxDXUFvAt04DeQ%3D%3D" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/14.0 Safari/605.1.15" "120.xx.xx.133" "127.0.0.1:9000" "response_location:https://innersso.xxxxxx.com?appid=star_admin&reqtoken=NDlMG0xy%2B%xxxxxx%2FcajkdRJ%2BP%xxxxxxx%2FqU%2FuAWvXDmhYWgaQlcZcrfq3Gm8YLfluQTF28Mxsz7B8s%2BiVCHy%2Bkij8yE%3D"
````

以上只是示例，在发送报警的时候需要考虑报警频率