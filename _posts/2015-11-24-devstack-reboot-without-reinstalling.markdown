---
layout: post
title:  "Devstack如何重新启动后保留原有Openstack环境"
date:   2015-11-24 17:53:00 +0800 
categories: openstack
---

# 我们要做什么
一般我们使用Devstack搭建Openstack的测试环境，但是我们发现devstack每次重启都会清空所有数据并重新安装所有服务，
这浪费了我们大量的时间， 而且重新安装服务还需要网络，在中国这常常是一个极为痛苦的过程。
devstack这么做的目的是认为虚拟机的环境会发生各种变化，所以它在每次重启时都会重新配置整个系统。
但是实际环境中，我们的环境常常并不会发生变化。 官方目前没有针对这种情况的解决方法，所以本文就是要提出这种问题的解决方法。



# 本文基于的环境
本文的环境如下
- 使用devstack部署K版的openstack, devstack的路径为/opt/stack/devstack
- hypervisor 使用的是 xenserver


# 我们如何配置

## 思路如何
我们需要禁止 stack.sh 在开机时运行，然后手动做一些本来应该在 stack.sh 中做的设置，
最后使用rejoin-stack.sh将其他的服务启动起来。


## 实际配置

首先禁止stack.sh做任何事情。

{% highlight bash %}
sed -i  '1a exit 0'  /opt/stack/devstack/stack.sh  # 禁止stack.sh做任何事情
{% endhighlight %}

然后重启系统，重启后按下面的脚本恢复服务。

{% highlight bash %}
cd /etc/apache2/sites-enabled && sudo ln -s ../sites-available/keystone.conf  .  # 重新启用 keystone 的apache 配置

# 重启各种服务
sudo service apache2 restart
sudo service rabbitmq-server restart
sudo service mysql restart
sudo service tgt restart

# 重新为cinder配置lvm
truncate -s 10737418240 /opt/stack/data/stack-volumes-lvmdriver-1-backing-file   # 设置10G空间用于cinder提供块存储
CINDER_DEVICE=`sudo losetup -f --show /opt/stack/data/stack-volumes-lvmdriver-1-backing-file`
sudo pvcreate $CINDER_DEVICE
sudo vgcreate  stack-volumes-lvmdriver-1 $CINDER_DEVICE

# 重新启用其他服务
cd /opt/stack/devstack/   && ./rejoin-stack.sh
{% endhighlight %}



# 还存在的问题
- 我们使用这套方法后， 能保留大部分我们的数据， 包括mysql中的数据和自己改的各种配置文件。 但是还是有一些数据丢失了
， 比如你在原来lvm中存储的数据。
- 一些我暂时不用的服务我没有去弄，比如glance中的某些服务用这种方法还未能正常启动。 但是实际配置应该也不复杂，
和上面脚本中配置LVM差不多。



# 参考资料
1. [开源社区针对这个问题的讨论][1]

[1]: http://lists.openstack.org/pipermail/openstack-dev/2015-November/080167.html        "开源社区针对这个问题的讨论"
