<div class="note inf">本文使用的 Zabbix 版本为 4.4.10，Smtp 服务器为 163 邮箱</div>

### I 配置 Mail

1. 安装 `mailx`

   `yum list installed mailx`

2. 编辑配置文件

   `vim /etc/mail.rc`

   ```shell
   # 发送人的邮箱
   set from=***@163.com
   # 邮箱服务器
   set smtp=smtp.163.com
   # 发送人的邮箱
   set smtp-auth-user=***@163.com
   # 授权码
   set smtp-auth-password=***
   set smtp-auth=login
   ```

3. 测试发送

   `echo` 是文章内容，`-s` 后面跟随文章标题和目标邮箱

   `echo "agent down" | mail -s "test mail" ***@qq.com`

### II 配置 Zabbix

1. 编写脚本

   查看 `zabbix_server.conf` 配置的 `AlertScriptsPath` 在哪个路径，如果没有则添加配置，报警脚本要放在该路径下。在 `/usr/lib/zabbix/alertscripts` 目录下，执行 `vim sendmail.sh`

   ```shell
   #!/usr/bin/bash
   messages=`echo $3 | tr '\r\n' '\n'`
   subject=`echo $2 | tr '\r\n' '\n'`
   echo "${messages}" | mail -s "${subject}" $1 >>/tmp/sendmail.log 2>&1
   ```

   一共三个参数 

   - $1：to 目标邮箱
   - $2：subject 邮件标题
   - $3：message 邮件内容

   编辑完需要改权限、用户和分组

   ```shell
   chown zabbix.zabbix sendmail.sh
   chmod 777 sendmail.sh
   ```

   

2. 配置 zabbix 媒介类型

   选择脚本类型并引用刚才的脚本，下图中的参数作为测试使用

   ![](https://cdn.jsdelivr.net/gh/mapleinsss/md_cdn@master/mds/zabbix/sendMail/1.png)

   配置完成后，点击更新后执行测试，如果发送成功，说明脚本配置成功。测试完成后，删除脚本参数，添加新参数，分别对应脚本中的三个参数

   ![](https://cdn.jsdelivr.net/gh/mapleinsss/md_cdn@master/mds/zabbix/sendMail/2.png)

   

3. 配置报警媒介用户

   配置发送的用户，需要配置上面的报警媒介类型，以及收件人等，按照下图配置即可，配置完成点击更新。

   ![](https://cdn.jsdelivr.net/gh/mapleinsss/md_cdn@master/mds/zabbix/sendMail/3.png)

4. 添加动作

   这里告警动作使用的触发器为 1 分钟 cpu 负载超过 30%。

   - 解释：触发器会监控某个指标，当数值超过该指标，触发器触发。如果在动作里配置了该触发器，那么触发器就会触发该动作的发生，动作可以定义发送邮件或其他操作。

   下图可以看到动作后面有三种操作类型：

   - 操作：触发器触发时执行；
   
   - 恢复操作：触发器监控指标恢复正常时执行；

   - 更新操作：问题发生更新时执行；

     ![](https://cdn.jsdelivr.net/gh/mapleinsss/md_cdn@master/mds/zabbix/sendMail/4.png)

   这里只对操作进行配置，后面两个操作基本一样：

   - 操作持续时间 1h，代表在一小时会发送一次邮件出去

   - 标题和内容的表达式，可以去网上找，下面给一个例子
   
     ```
     Problem ! CPU Average GT 30%: [{HOSTNAME1}]:{TRIGGER.NAME}
     
     告警时间:  {EVENT.DATE} {EVENT.TIME}
     告警信息:  {TRIGGER.NAME}
     告警主机:  {HOST.NAME}
     告警级别:  {TRIGGER.SEVERITY}
     告警项目:  {TRIGGER.KEY1}
     
     问题详情:  {ITEM.NAME}:{ITEM.VALUE}
     当前状态:  {TRIGGER.STATUS}
     
     Original problem ID: {TRIGGER.ID}
     ```
     
   - 细节内容则是配置持续时间，操作类型和接收告警的人等
   
     ![](https://cdn.jsdelivr.net/gh/mapleinsss/md_cdn@master/mds/zabbix/sendMail/5.png)



### III 测试

下面将进行测试，验证之前的配置。

1. 输入命令，让 cpu 动起来，执行后用 `top` 查看 cpu 占用率。

   ```shell
   for i in `seq 1 $(cat /proc/cpuinfo |grep "physical id" |wc -l)`; do dd if=/dev/zero of=/dev/null & done
   ```

2. 配置过程中选用的触发器是每分钟统计一次 cpu 负载，当大于 30% 时发送告警邮件，等待一段时间，可以在邮箱收到告警内容如下：

   ![](https://cdn.jsdelivr.net/gh/mapleinsss/md_cdn@master/mds/zabbix/sendMail/6.png)

3. 结束掉之前跑 cpu 的命令

   ```shell
   pkill -9 dd
   ```

4. 等待一分钟，会收到恢复的邮件

   ![](https://cdn.jsdelivr.net/gh/mapleinsss/md_cdn@master/mds/zabbix/sendMail/7.png)

   

### IV 总结

通过对 Zabbix 配置，实现了当触发器触发时执行发送邮件的动作，对系统运维人员进行通知，从而第一时间对系统问题进行排查和修复。

Zabbix 是一套强大的免费开源监控解决方案，要学的东西还很多！