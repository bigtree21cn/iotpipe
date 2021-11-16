# iotpipe
device(json)-> mqtt -> telegraf -> influxdb


#总体架构:
	IoT Simulator(publisher)----> MQTT broker---->Telegraf(subscriber)---->InfluxDB----> Grafana
	
	客户端上报topic: myproduct/mydevice//thing/event/property/post
	物模型 - 属性
	{
		"id": 23239,
		"param": {
			"temperature": 25,
			"pm25": 72,
			"co2": 300
		},
		"method": "thing.event.property.post"
	}
	
	
# IoT Simulator:
		○ 使用的是MQTTBox （好像要把slack proxy关闭，因为local地址也代理了？）
		Download link (windows)
		○ 可以尝试产生随机jason数据 IoT Simulator: GitHub - everwatchsolutions/json-data-generator: A robust, generic, streaming random json data generator for your data
	
# 	Mosquito (MQTT Broker):
		○ 直接写的deployment.yaml, 包括pod/service/configmap, service使用NodePort方便外部调试， 当作一个broker转发数据，没有持久化配置。
		○ 把配置的log的信息，比如日志等级，连接建立等配置齐全，方便调试。配在: /mosquitto/log/
	
	
# 	NFS server provisioner:
	方便持久化数据, 配有一个NFS server， 以及nfs provisoner和预设的storageClass "nfs"
	
	helm repo add stable https://charts.helm.sh/stable --force-update
	helm repo add kvaps https://kvaps.github.io/charts
	helm install my-nfs-server-provisioner kvaps/nfs-server-provisioner --version 1.3.1
	
		○ [注意： 在 /etc/systemd/system/docker.service.d/http-proxy.conf 设置了docker 到本地的slack代理]
		○ 我修改了template/statefulset.yaml ，把数据存储目录export对应的volumn从emptydir变为hostPath，并且把value.yaml中的nodeSelector定义为node01上。
		○ Pod mount不上nfs, 原因是缺少 /sbin/mounts.nfs 驱动，我在每个节点上安装了 apt install nfs-common， 

# 	InfluxDB:
	helm repo add influxdata https://helm.influxdata.com/
	Helm repo update
	helm install my-influxdb influxdata/influxdb --version 4.10.0
		○ 记得把persistent相关参数该了，主要是size和StorageClass=nfs.  这一步依赖上一步nfs的配置
	
	
	
# 	Telegraf:
	Helm install my-telegraf influxdata/telegraf 
	
	一些调试telegraf的tips:
		○ 记得把telegraf.conf 的debug打开
		○ 使用多个input来调试。 比如 inputs.file + inputs.mqtt_consumer的udp模式(通过nc命令发送你的输入), 这样同样的输入可以有不一样的配置，可以对比输出结果。
		○ 对output，可以使用outputs.influxdb的udp模式进行输出镜像，然后通过nc -up xxxportxxx 来实时看输出。同样的，可以对比多个output stream来调试输出配置。
	
# 	Grafana:   <host安装，非容器>
	apt install grafana
	systemctl start grafana-server
	
# 	Test:
	http://localhost:3000/goto/ztOSk3Vnz?orgId=1
![image](https://user-images.githubusercontent.com/16154205/141928555-2850822a-666b-4571-823f-bf4f83301496.png)
