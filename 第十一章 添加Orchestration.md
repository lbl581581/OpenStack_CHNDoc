
# 11. 添加服务流程  #<br>
目录<br>
服务流程回顾 ...................................................................................... 91<br>
安装业务流程服务 ..................................................................................... 91<br>
验证服务流程安装 ................................................................... 93<br>
下一步 ..................................................................................................................... 95<br>
靠HOT的模板语言通过业务流程的模板来创造云资源. 集成项目的名字是heat.
## 服务流程回顾 ##
这服务流程提供了一个最基本的模板流程来描述一个通过运行OpenStack API来调用运行的云应用.微软将OpenStack其它的核心部件集成进单文件模板系统，这些模板能让创造大部分类型的OpenStack资源.
比如动态IP，卷，安全组，用户等等，此外还提供了一些先进的功能，比如高可用性，例如自动缩放，和嵌套栈.通过提供非常紧密地集成其他OpenStack的核心项目，OpenStack核心
项目可以得到一个更大的用户群.该服务能够直接部署整合服务流程或者通过自定义插件.
 
服务流程由以下几点组成：<br>
• 命令行客户端. 一个命令界面通过操控API来运行自动气象站CloudFormation API.最后开发商也可以直接使用业务流程API<br>
• heat-api组件. 提供了一个本地的OpenStack的API以至于进程API需要通过发送它们给heat-engine而不再使用远程调用 <br>
• heat-api-cfn组件. 提供一个自动气象站查询API以致和自动气象站CloudFormation相协调以及进程API发送请求给heat-engine而不再使用远程调用 <br>
• heat-engine. 流程模板的运行和提供事件给API使用者 <br>
# 安装业务流程服务 <br>
1. 安装业务流程服务在控制节点上:<br>
		 
		# yum install openstack-heat-api openstack-heat-engine \ openstack-heat-api-cfn <br>
2. 在配置文件中，服务流程将数据存储在数据库特别的地方.有一些例子就使用MYSQL数据库将heat上的用户置于控制节点上.用HEAT_DBPASS取代用户密码： 
		
		#openstack-config --set /etc/heat/heat.conf \ 
		database connection mysql://heat:HEAT_DBPASS@controller/heat
3. 使用之前你在日志里设定好的管理员密码来创建用户插入heat数据库 <br>
		
		$ mysql -u root -p
		mysql> CREATE DATABASE heat;
		mysql> GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'localhost' \
		IDENTIFIED BY 'HEAT_DBPASS';<br>
		mysql> GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%' \
		IDENTIFIED BY 'HEAT_DBPASS';
4. 创建heat服务表格:<br>
 
		#su -s /bin/sh -c "heat-manage db_sync" heat
注<br>
忽略 DeprecationWarning 错误.
5. 配置服务流程使用QPID消息代理:

		#openstack-config --set /etc/heat/heat.conf DEFAULT
		qpid_hostname controller
6. 创建heat用户以致服务流程可以使用的鉴定与身份识别服务.使用权限给予服务给用户以管理权限：

		$ keystone user-create --name=heat --pass=HEAT_PASS \
		--email=heat@example.com
		$ keystone user-role-add --user=heat --tenant=service --role=admin
7. 运行以下命令来配置服务流程进行身份验证与身份服务:

		# openstack-config --set /etc/heat/heat.conf keystone_authtoken \
		auth_uri http://controller:5000/v2.0
		# openstack-config --set /etc/heat/heat.conf keystone_authtoken \
		auth_port 35357
		# openstack-config --set /etc/heat/heat.conf keystone_authtoken \
		auth_protocol http
		# openstack-config --set /etc/heat/heat.conf keystone_authtoken \
		admin_tenant_name service
		# openstack-config --set /etc/heat/heat.conf keystone_authtoken \
		admin_user heat
		# openstack-config --set /etc/heat/heat.conf keystone_authtoken \
		admin_passwordHEAT_PASS
		# openstack-config --set /etc/heat/heat.conf ec2authtoken \
		auth_uri http://controller:5000/v2.0
8. 用身份识别服务寄存Heat 和CloudFormation APIs 以致于其它的OpenStack服务可以放置那些APIs. 寄存服务和特别的端口:

		$ keystone service-create --name=heat --type=orchestration \
		--description="Orchestration"
		$ keystone endpoint-create \
		--service-id=$(keystone service-list | awk '/ orchestration / {print
		$2}') \
		--publicurl=http://controller:8004/v1/%\(tenant_id\)s \
		--internalurl=http://controller:8004/v1/%\(tenant_id\)s \
		--adminurl=http://controller:8004/v1/%\(tenant_id\)s
		$ keystone service-create --name=heat-cfn --type=cloudformation \
		--description="Orchestration CloudFormation"
		$ keystone endpoint-create \
		--service-id=$(keystone service-list | awk '/ cloudformation / {print
		$2}') \
		--publicurl=http://controller:8000/v1 \
		--internalurl=http://controller:8000/v1 \
		--adminurl=http://controller:8000/v1
9. 创建heat_stack_user角色.这角色被认作默认的对于用户由流程模板创建.运行以下指令来创建heat_stack_user角色

		$ keystone role-create --name heat_stack_user
10. 配置元数据和等待服务器返回URL.运行以下命令来修改默认节点下/etc/heat/heat.conf文件：

		# openstack-config --set /etc/heat/heat.conf \
		DEFAULT heat_metadata_server_url http://10.0.0.11:8000
		# openstack-config --set /etc/heat/heat.conf \
		DEFAULT heat_waitcondition_server_url http://10.0.0.11:8000/v1/
		waitcondition
注<br>
这个例子使用控制器的IP(10.0.0.11)代替了控制器的主机名字因此我们的架构不包括DNS设置. 如果你选择将它放入URL中要确保这实例能解决这控制器的主机名字.
11. 当系统重启时开始heat-api, heat-api-cfn and heat-engine服务和配置它们开始:

		# service openstack-heat-api start
		# service openstack-heat-api-cfn start
		# service openstack-heat-engine start
		# chkconfig openstack-heat-api on
		# chkconfig openstack-heat-api-cfn on
		# chkconfig openstack-heat-engine on
# 验证业务流程服务安装 #<br>
验证业务流程服务是否正确安装和配置，确保你的凭据在demo-openrc.sh文件设置正确.源文件如下指令:<br>
		
		$ source demo-openrc.sh
业务流程模块使用模板描述栈.了解模板语言，去看模板引导在Heat的开发文档里.以下内容创建测试模板在test-stack.yml文件里：

		heat_template_version: 2013-05-23
		description: Test Template
		parameters:
		ImageID:
		type: string
		description: Image use to boot a server
		NetID:
		type: string
		description: Network ID for the server
		resources:
		server1:
		type: OS::Nova::Server
		properties:
		name: "Test server"
		image: { get_param: ImageID }
		flavor: "m1.tiny"
		networks:
		- network: { get_param: NetID }
		outputs:
		server1_private_ip:
		description: IP address of the server in the private network
		value: { get_attr: [ server1, first_address ] }
从模板中使用heat栈创建指令来创建栈：<br>

		$ NET_ID=$(nova net-list | awk '/ demo-net / { print $2 }')
		$ heat stack-create -f test-stack.yml \
		-P "ImageID=cirros-0.3.2-x86_64;NetID=$NET_ID" testStack
		+--------------------------------------+------------+--------------------
		+----------------------+
		| id | stack_name | stack_status |
		creation_time |
		+--------------------------------------+------------+--------------------
		+----------------------+
		| 477d96b4-d547-4069-938d-32ee990834af | testStack | CREATE_IN_PROGRESS |
		2014-04-06T15:11:01Z |
		+--------------------------------------+------------+--------------------
		+----------------------+
验证着栈被成功的创建通过heat stack-list指令:<br>

		$ heat stack-list
		+--------------------------------------+------------+-----------------
		+----------------------+
		| id | stack_name | stack_status |
		creation_time |
		+--------------------------------------+------------+-----------------
		+----------------------+
		| 477d96b4-d547-4069-938d-32ee990834af | testStack | CREATE_COMPLETE |
		2014-04-06T15:11:01Z |
		+--------------------------------------+------------+-----------------
		+----------------------+
下一步<br>
你OpenStack环境包含业务流程.你可以发起一个实例或者更多的服务对你的环境在以下章节中.