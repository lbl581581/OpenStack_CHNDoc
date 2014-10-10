
# 12. 添加遥测模块 #<br>
目录<br>
遥测 ...................................................................................................................... 96<br>
安装遥测模块 ......................................................................................... 97<br>
安装遥测计算代理 ....................................................................... 99<br>
遥测图像服务配置 .................................................................. 100<br>
为遥测添加块存储服务代理 ...................................................... 101<br>
为遥测配置对象存储服务系统 ..................................................... 101<br>
验证遥测安装 .................................................................................. 102<br>
下一步 ................................................................................................................... 103<br>
遥测为监测计量OpenStack云提供了一个框架.它也被称为云项目.
# 遥测 #
遥测模块:<br>
• 有效地收集计量数据关于CPU和网络成本.<br>
• 收集数据通过监测从服务器发送通知或通过投票设施服务.<br>
• 配置不同类型的数据来满足各种各样操作要求.访问和插入计量数据通过其他API.<br>
• 扩展框架通过附加插件来收集常使用数据.<br>
• 产生不能拒绝签署的计量信息.<br>
该系统由以下几个基本部件组成:<br>
• 计算代理（云代理计算）.运行在每个计算节点和对资源利用统计调查.有可能在未来的其他类型的代理.但现在我们将重点放在建立计算代理.<br>
• 一个中央代理（云代理中心）.运行在一个中央管理服务器对不依赖于实例或计算节点的资源利用统计调查.<br>
• 集电极（云幂仪器）.运行在一个或多个中央管理服务器监控消息队列（从代理通知和计量数据）.通知消息的处理和转化为计量消息返回到使用适当的主题的消息总线.遥测信息写入数据存储没有修改.<br>
• 警告通知（云幂仪报警通知）.运行在一个或多个中央管理服务器允许设置警告基于阈值评价收集的样品.<br>
• 数据存储。数据库能够处理并行写入（从一个或更多的收集器实例）和读取（从服务器的API）.<br>
• 一个API服务器（云API）.运行在一个或多个中央管理服务器提供从数据存储中的数据.<br>
这些服务交流通过使用标准的OpenStack消息总线通信。只有收集器和API服务器能够访问数据存储.<br>
# 安装遥测模块 #
遥测技术通过一个API服务来提供了一个集热器和一系列不同的代理.你可以在节点如计算节点安装这些代理，你必须用这个程序来安装核心组件在控制器节点上.<br>
1. 在控制器节点安装遥测服务:

		# yum install openstack-ceilometer-api openstack-ceilometer-collector \
		openstack-ceilometer-notification openstack-ceilometer-central
		openstack-ceilometer-alarm \
		python-ceilometerclient
2.遥测服务使用数据库来存储信息.在数据库指定位置的配置文件中.该示例使用MongoDB数据库上在控制器节点上:
		
		# yum install mongodb-server mongodb
注<br>
MongoDB数据库的默认配置是在/var/lib/mongodb/journal/直接目录下创建多个1GB的文件来保存数据库日志.<br>
如果你想要降低日志的分配空间，将在/etc/mongodb.conf下的配置文件里的关键字设为true.这个配置设置能使每个日志文件的大小减少到512MB.<br>
更多的信息有关配置设置请参考MongoDB文档http://docs.mongodb.org/manual/reference/configuration-options/#smallfiles.<br>
用于指示禁用数据库日志完全步骤详细说明参考http://docs.mongodb.org/manual/tutorial/manage-journaling/.<br>
3. 配置MongoDB使它监听控制器管理IP地址。编辑/etc/mongodb.conf文件和修改bind_ip:

		bind_ip = 10.0.0.11
4.当系统系统启动时启动MongoDB服务器并配置它:

		# service mongod start
		# chkconfig mongod on
5.创建数据库和云数据库用户:

		# mongo --host controller --eval '
		db = db.getSiblingDB("ceilometer");
		db.addUser({user: "ceilometer",
		pwd: "CEILOMETER_DBPASS",
		roles: [ "readWrite", "dbAdmin" ]})'
6.配置遥测服务使用的数据库:

		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		database connection mongodb://
		ceilometer:CEILOMETER_DBPASS@controller:27017/ceilometer
7.必须定义作为一个共享的密钥在遥测服务节点。使用OpenSSL来生成一个随机的令牌并将其存储在配置文件:

		# CEILOMETER_TOKEN=$(openssl rand -hex 10)
		# echo $CEILOMETER_TOKEN
		# openstack-config --set /etc/ceilometer/ceilometer.conf publisher
		metering_secret $CEILOMETER_TOKEN
8.配置QPID访问:

		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		DEFAULT rpc_backend ceilometer.openstack.common.rpc.impl_qpid
9.创建一个用户，云高仪遥测服务使用的鉴定与身份识别服务。使用服务租户和给用户管理员角色:

		$ keystone user-create --name=ceilometer --pass=CEILOMETER_PASS --
		email=ceilometer@example.com
		$ keystone user-role-add --user=ceilometer --tenant=service --role=admin
10.配置遥测服务进行身份验证的身份服务.设置auth_strategy的值在 /etc/ceilometer/ceilometer.conf 文件里:

		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		DEFAULT auth_strategy keystone
11.添加凭据到配置文件的遥测服务:

		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		keystone_authtoken auth_host controller
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		keystone_authtoken admin_user ceilometer
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		keystone_authtoken admin_tenant_name service
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		keystone_authtoken auth_protocol http
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		keystone_authtoken auth_uri http://controller:5000
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		keystone_authtoken admin_password CEILOMETER_PASS
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		service_credentials os_auth_url http://controller:5000/v2.0
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		service_credentials os_username ceilometer
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		service_credentials os_tenant_name service
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		service_credentials os_password CEILOMETER_PASS
12.寄存器遥测服务通过身份识别服务使的其它OpenStack可以存储.使用关键命令来注册服务和指定端点:

		$ keystone service-create --name=ceilometer --type=metering \
		--description="Telemetry"
		$ keystone endpoint-create \
		--service-id=$(keystone service-list | awk '/ metering / {print $2}') \
		--publicurl=http://controller:8777 \
		--internalurl=http://controller:8777 \
		--adminurl=http://controller:8777
13.当系统启动时开始openstack-ceilometer-api, openstack-ceilometer-central,openstack-ceilometer-collector以及服务和配置:

	# service openstack-ceilometer-api start
	# service openstack-ceilometer-notification start
	# service openstack-ceilometer-central start
	# service openstack-ceilometer-collector start
	# service openstack-ceilometer-alarm-evaluator start
	# service openstack-ceilometer-alarm-notifier start
	# chkconfig openstack-ceilometer-api on
	# chkconfig openstack-ceilometer-notification on
	# chkconfig openstack-ceilometer-central on
	# chkconfig openstack-ceilometer-collector on
	# chkconfig openstack-ceilometer-alarm-evaluator on
	# chkconfig openstack-ceilometer-alarm-notifier on
# 安装遥测计算代理 #
遥测技术通过了一个API服务来提供了一个收集器和一系列不同的代理.<br>
本程序详细介绍了如何安装运行在计算节点代理.<br>
1. 在计算节点上安装遥测服务:

		# yum install openstack-ceilometer-compute python-ceilometerclient pythonpecan
2.进行以下设置在/etc/nova/nova.conf 文件中:

		# openstack-config --set /etc/nova/nova.conf DEFAULT \
		instance_usage_audit True
		# openstack-config --set /etc/nova/nova.conf DEFAULT \
		instance_usage_audit_period hour
		# openstack-config --set /etc/nova/nova.conf DEFAULT \
		notify_on_state_change vm_and_task_state
注<br>
该notification_driver的选择是一个多值的选择, 因此openstack-config 无法设定值. 参考章节 “OpenStackpackages” [19].<br>
编辑目录下/etc/nova/nova.conf 文件和添加行对于这默认的选择:

		[DEFAULT]
		...
		notification_driver = nova.openstack.common.notifier.rpc_notifier
		notification_driver = ceilometer.compute.nova_notifier
		rpc_backend = qpid
3.重新启动计算机服务:

		# service openstack-nova-compute restart
4.你必须在你定义之前设定密钥.遥测服务节点分享这个密钥作为一个共享的密钥:

		# openstack-config --set /etc/ceilometer/ceilometer.conf publisher \
		metering_secret CEILOMETER_TOKEN
5.配置QPID访问:

		# openstack-config --set /etc/ceilometer/ceilometer.conf DEFAULT
		rpc_backend ceilometer.openstack.common.rpc.impl_qpid
		# openstack-config --set /etc/ceilometer/ceilometer.conf DEFAULT
		qpid_hostname controller
6.添加的身份服务凭据:
		
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		keystone_authtoken auth_host controller
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		keystone_authtoken admin_user ceilometer
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		keystone_authtoken admin_tenant_name service
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		keystone_authtoken auth_protocol http
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		keystone_authtoken admin_password CEILOMETER_PASS
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		service_credentials os_username ceilometer
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		service_credentials os_tenant_name service
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		service_credentials os_password CEILOMETER_PASS
		# openstack-config --set /etc/ceilometer/ceilometer.conf \
		service_credentials os_auth_url http://controller:5000/v2.0
7. 系统启动时启动服务并将其配置为启动时:

		# service openstack-ceilometer-compute start
		# chkconfig openstack-ceilometer-compute on
# 遥测图像服务配置 #
1. 检索图像样本，你必须配置图像服务发送通知总线.运行以下命令:

		# openstack-config --set /etc/glance/glance-api.conf DEFAULT
		notification_driver messaging
		# openstack-config --set /etc/glance/glance-api.conf DEFAULT rpc_backend
		qpid
2. 他们的新设置重新启动图像服务:

		# service openstack-glance-api restart
		# service openstack-glance-registry restart
# 添加遥测块存储服务代理 #
1. 为了检索样品的量你必须配置块存储服务通知发送到总线.运行以下命令在控制器节点和体积节点:

		# openstack-config --set /etc/cinder/cinder.conf DEFAULT control_exchange
		cinder
		# openstack-config --set /etc/cinder/cinder.conf DEFAULT
		notification_driver cinder.openstack.common.notifier.rpc_notifier
2. 他们的新设置重新启动块存储服务.在这控制节点上:

		# service openstack-cinder-api restart
		# service openstack-cinder-scheduler restart
在体积节点上:

		# service openstack-cinder-volume restart
# 为遥测配置对象存储服务 #
1. 检索对象存储统计，遥测服务需要管理员权限来访问对象存储.给这个角色你os_username用户为os_tenant_name租户:

		$ keystone role-create --name=ResellerAdmin
		+----------+----------------------------------+
		| Property | Value |
		+----------+----------------------------------+
		| id | 462fa46c13fd4798a95a3bfbe27b5e54 |
		| name | ResellerAdmin |
		+----------+----------------------------------+
		$ keystone user-role-add --tenant service --user ceilometer \
		--role 462fa46c13fd4798a95a3bfbe27b5e54
2. 你还必须添加遥测中间件对象存储处理传入的和传出的流量. 添加这些线到/etc/swift/proxy-server.conf 文件:

		[filter:ceilometer]
		use = egg:ceilometer#swift
3. 添加云幂仪的管道参数到同一文件:

		[pipeline:main]
		pipeline = healthcheck cache authtoken keystoneauth ceilometer proxyserver
4. 随着新的设置，重新启动服务:

		# service openstack-swift-proxy restart
# 验证遥测装置 #
测试遥测装置，从图像服务下载图像，并使用云幂仪命令显示统计.<br>
1. 使用云高仪表列表命令测试遥测的访问:

		$ ceilometer meter-list
		+------------+-------+-------+--------------------------------------
		+---------+----------------------------------+
		| Name | Type | Unit | Resource ID | User
		ID | Project ID |
		+------------+-------+-------+--------------------------------------
		+---------+----------------------------------+
		| image | gauge | image | acafc7c0-40aa-4026-9673-b879898e1fc2 | None
		| efa984b0a914450e9a47788ad330699d |
		| image.size | gauge | B | acafc7c0-40aa-4026-9673-b879898e1fc2 | None
		| efa984b0a914450e9a47788ad330699d |
		+------------+-------+-------+--------------------------------------
		+---------+----------------------------------+
2.从图像服务下载图像:

		$ glance image-download "cirros-0.3.2-x86_64" > cirros.img
3.调用云幂仪表列表命令再次验证，下载已检测到的遥测数据存储:

		$ ceilometer meter-list
		+----------------+-------+-------+--------------------------------------
		+---------+----------------------------------+
		| Name | Type | Unit | Resource ID |
		User ID | Project ID |
		+----------------+-------+-------+--------------------------------------
		+---------+----------------------------------+
		| image | gauge | image | acafc7c0-40aa-4026-9673-b879898e1fc2 |
		None | efa984b0a914450e9a47788ad330699d |
		| image.download | delta | B | acafc7c0-40aa-4026-9673-b879898e1fc2 |
		None | efa984b0a914450e9a47788ad330699d |
		| image.serve | delta | B | acafc7c0-40aa-4026-9673-b879898e1fc2 |
		None | efa984b0a914450e9a47788ad330699d |
		| image.size | gauge | B | acafc7c0-40aa-4026-9673-b879898e1fc2 |
		None | efa984b0a914450e9a47788ad330699d |
		+----------------+-------+-------+--------------------------------------
		+---------+----------------------------------+
4.你现在可以为各种得到使用统计:

		$ ceilometer statistics -m image.download -p 60
		+--------+---------------------+---------------------+-------
		+------------+------------+------------+------------+----------
		+----------------------------+----------------------------+
		| Period | Period Start | Period End | Count | Min
		| Max | Sum | Avg | Duration | Duration Start
		| Duration End |
		+--------+---------------------+---------------------+-------
		+------------+------------+------------+------------+----------
		+----------------------------+----------------------------+
		| 60 | 2013-11-18T18:08:50 | 2013-11-18T18:09:50 | 1 | 13167616.0
		| 13167616.0 | 13167616.0 | 13167616.0 | 0.0 | 2013-11-18T18:09:05.
		334000 | 2013-11-18T18:09:05.334000 |
		+--------+---------------------+---------------------+-------
		+------------+------------+------------+------------+----------
		+----------------------------+----------------------------+
下一步
你现在OpenStack环境包括遥测.你可以启动一个实例或增加更多的服务在你前面的章节中的环境.