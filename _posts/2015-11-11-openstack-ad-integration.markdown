---
layout: post
title:  "如何使用Keystone用AD域作为后端"
date:   2015-11-11 16:53:00 +0800 
categories: openstack
---

# 我们要做什么
一般在Openstack中，用户的认证是由keystone模块实现， keystone的后台存储一般是MySQL、Postgrel这种数据库。
但是有时候你不能使用这些数据库，比如需求里要求你直接使用windows AD域里的账户直接登录Openstack。
本文要讲的就是在这种情景下，如何配置keystone。


# 基本概念

## AD域是什么
AD域是一种机制，它是实现一套集中的账号和权限管理机制。在AD域中具体的权限管理是由域控服务器(DC, Domain Controllers)实现，域控服务器存储了所有的用户信息、域内的计算机信息和权限信息。
当一台机器加入到域控服务器时，那么域内的所有用户就可以登录这台机器，并且可以通过这台机器访问域内所有他有权限的资源。
[这个视频][4]对AD域的讲解非常详细，[win server 2012配置域控服务器系列教程][3]讲解了如何配置DC。


LDAP和AD域是什么关系呢？ 

- LDAP叫Lightweight Directory Access Protocol，它其实是一种协议，这套协议了一种树状的存储机制，然后定义了访问存储的内容的方式。
- AD域其实就是LDAP的一种实现，其实你需要知道的是，AD域的那些用户、权限都是通过树状结构存储的。


# 本文基于的环境
本文已经在一台机器上使用devstack部署好了K版的openstack，ip为 192.168.1.113。
使用windows server 2012作为AD域的DC服务器，ip为 192.168.1.106。



# 我们如何配置

## 思路如何
最靠谱的文档是代码里的官方文档, 你在源码的`keystone/doc/source/configuration.rst`中可以看到最新的如何集成keystone和ldap的文档，我摘录一小段


{% highlight ReST %}
Read Only LDAP
--------------

Many environments typically have user and group information in directories that
are accessible by LDAP. This information is for read-only use in a wide array
of applications. Prior to the Havana release, we could not deploy Keystone with
read-only directories as backends because Keystone also needed to store
information such as projects, roles, domains and role assignments into the
directories in conjunction with reading user and group information.

Keystone now provides an option whereby these read-only directories can be
easily integrated as it now enables its identity entities (which comprises
users, groups, and group memberships) to be served out of directories while
resource (which comprises projects and domains), assignment and role
entities are to be served from different Keystone backends (i.e. SQL). To
enable this option, you must have the following ``keystone.conf`` options set:

.. code-block:: ini

  [identity]
  driver = keystone.identity.backends.ldap.Identity

  [resource]
  driver = keystone.resource.backends.sql.Resource

  [assignment]
  driver = keystone.assignment.backends.sql.Assignment

  [role]
  driver = keystone.assignment.role_backends.sql.Role
{% endhighlight %}


在文档中我们可以知道，在keystone中，我们是用 identity, resource,  assignment, role 这几种东西来描述一个用户的权限的。

- identity描述用户认证信息
- resource描述具体的资源，可能是一个tanent, domain等等。
- role描述具体的权限，比如时admin还是 member。
- assignment描述权限的分配，比如哪个用户(对应identity)对什么resource拥有什么样的权限(对应到role)，是一个多对多的关系。


在keystone里，这4中信息都可以分别设置后台存储时什么。


那么我们的思路就是，用户认证（也就是identity部分）使用ldap， 其他的使用原来的配置(在这里相当于使用devstack默认使用的mysql)。


达到的效果就是用户登录的时候会去查询ldap里的数据。具体有哪些tanent，权限怎么分配的，都按原来的数据库里的配置进行。



## 具体配置

讲清楚思路了，具体配置就很简单了。


配置keystone的Identity后台使用ldap driver
{% highlight ini %}
[identity]
# driver = keystone.identity.backends.sql.Identity
driver = keystone.identity.backends.ldap.Identity
{% endhighlight %}


配置ldap部分，即如何去找ldap中的数据
{% highlight ini %}
[ldap]
url = ldap://192.168.1.106
user = cn=admin,cn=Users,dc=young,dc=com
password = YOURPASSWORD
suffix = dc=young,dc=com
use_dumb_member = True
dumb_member = cn=OpenstackISCAS,cn=Users,dc=young,dc=com

user_tree_dn = cn=Users,dc=young,dc=com
user_objectclass = organizationalPerson
user_id_attribute = extensionName
user_name_attribute = sAMAccountName
user_mail_attribute = email
user_enabled_attribute = userAccountControl
user_enabled_mask = 2
user_enabled_default = 512
user_attribute_ignore = password,tenant_id,tenants
user_allow_create = True
user_allow_update = True
user_allow_delete = True
{% endhighlight %}

大致解释一下某几个属性的意思，具体的解释可以自己去代码中的文档里查看详细信息。

- `user_tree_dn`: 表示在AD域的存储目录中用户存储在哪里
- `user_objectclass`: 在`user_tree_dn`设置的目录里，设置只有这种类的对象才会被当成是用户
- `user_*_attribute`: 这几个属性意思就是告诉keystone设置AD域的时候，像id, name, mail 这种属性对应到 keystone中是什么。


下一步就是去AD域中按着keystone user表中原有的数据结构来配置数据。


比如我的keystone中原来有admin, glance, cinder等等这几个用户，然后我就可以去AD域中配置相应用户。


具体的配置可以使用AD域的用户管理工具，也可以使用 ADSI 或者 Softrerra LDAP Administrator来辅助配置。具体的流程繁琐但不困难，这里就不多说了。
(具体的配置方法可以参考[Keystone与LDAP协议集成][1]这篇文档)


比如我按我的方法配置后效果下图，需要注意的地方是用户id和username必须和原来系统中的一致，否则可能出现找不到权限的问题。
具体id用哪个属性，我这里是自己选择的。
![ANSI_config]({{site.baseurl}}/img/ldap_ANSI_config.jpg "image title")


# 还存在的问题
按本文方法设置ldap作为keystone的identity的存储后台后， 目前还不能通过keystone或者horizon对用户进行修改。



# 你可能会有的疑惑
你可能会搜索到下面的文档

- [Keystone与LDAP协议集成] [1]
- [官方说的如何集成][2]

然后你发现配置方法和本文提到的配置方法不一样。  其实这两篇是针对早期版本的Openstack进行设置的， 那时还不支持将resource, identity, role, assignment 这些资源分开backend进行存储。  本文只将identity改为使用ldap作为后端存储。



# 参考资料
1. [Keystone与LDAP协议集成][1]
1. [官方说的如何集成][2]
1. [win server 2012配置域控服务器系列教程][3]
1. [AD域的讲解视频][4]

[1]: http://blog.sina.com.cn/s/blog_6de3aa8a0101q4cb.html        "Keystone与LDAP协议集成"
[2]: https://wiki.openstack.org/wiki/HowtoIntegrateKeystonewithAD        "官方说的如何集成"
[3]: http://blog.csdn.net/xiezuoyong/article/details/9423759 "win server 2012配置域控服务器系列教程"
[4]: https://www.youtube.com/watch?v=lFwek_OuYZ8 "AD域的讲解视频"
