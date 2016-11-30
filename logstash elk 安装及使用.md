# logstash elk 安装及使用

标签： logstash elk 

---

## logstash 安装

环境要求: java > 7

安装java: `apt-get install openjdk-8-jre`

使用 apt 安装 logstash:
```
wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | apt-key add -
echo "deb https://packages.elastic.co/logstash/2.3/debian stable main" | tee -a /etc/apt/sources.list
apt-get update
apt-get install logstash
```
使用 yum 安装 logstash:
```
rpm --import https://packages.elastic.co/GPG-KEY-elasticsearch
```
> 将如下内容写入`/etc/yum.repos.d/logstash.repo`
```
[logstash-2.3]
name=Logstash repository for 2.3.x packages
baseurl=https://packages.elastic.co/logstash/2.3/centos
gpgcheck=1
gpgkey=https://packages.elastic.co/GPG-KEY-elasticsearch
enabled=1
```
> 执行`yum install logstash`

## logstash 运行

安装完成后的 logstash 执行文件位于 /opt/logstash/bin 下.

### 最简化运行方式
使用 logstash 可执行文件,运行如下命令:
`logstash -e 'input{stdin{}} output{elasticsearch{hosts=>"192.168.4.240"}}'`

此命令将使用标准输入做为 log 源, 输出到 elasticsearch 服务器上, 服务器地址由 hosts 定义.

执行此命令后, 命令会阻塞等待输入;此时输入内容后,将能够在 kibana 上查看到输入内容.

### 监控文件及修改提交内容
执行命令如下:
`logstash -e 'input{ file{path=>"/proc/13899/root/var/log/nginx/test.log"} } filter{mutate{ replace => {"path" => "/var/log/nginx/test.log" "host"=>"2da0903ff372"} } }  output{elasticsearch{hosts=>"192.168.4.240"}}'`

监控了 `/proc/13899/root/var/log/nginx/test.log` 文件并在提交时,将显示路径修改为 `/var/log/nginx/test.log`,添加了 host 值为 2da0903ff372.

## elk 容器化安装

执行命令:
`docker run -itd -p 5601:5601 -p 9200:9200 -p 5044:5044 -p 5000:5000 --name elk sebp/elk`

以容器化方式运行elk. 通过 ip:5601 访问 kibana. 在有若干日志后可确认 pattern 开始查看.可以使用 logstash 的最简化运行方式来输入初始的日志.

或通过如下命令使用官方容器:
`docker run --name es -d -p 9200:9200 -p 9300:9300 elasticsearch`
`docker run --name kibana -e ELASTICSEARCH_URL=http://192.168.4.240:9200 -p 5601:5601 -d kibana`



