# 实现cas ticket基于redis的集群


### 实现思路：
     将cas中的ticket票据存放到 redis缓存中，实现多个cas节点获取ticket的时候从一个地方获取。

### 实现步骤：

* 添加依赖  **cas-server-core**模块的 pom.xml

		<!-- redis -->
        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-redis</artifactId>
            <version>1.5.1.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
            <version>2.2</version>
        </dependency>
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.6.2</version>
        </dependency>

* 修改 spring配置文件 **cas-server-webapp** 模块 WEB-INF\spring-configuration\ticketRegistry.xml

		
		<bean id="ticketRegistry" class="org.jasig.cas.ticket.registry.DefaultTicketRegistry" />

	替换为

		<bean id="ticketRegistry" class="org.jasig.cas.ticket.registry.RedisTicketRegistry"
          p:redisTemplate-ref="redisTemplate"/>

	    <bean id="jedisConnFactory"          			class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
	          p:hostName="182.92.4.161"
	          p:database="1"
	          p:usePool="true"/>
	
	    <bean id="redisTemplate" class="org.jasig.cas.ticket.registry.TicketRedisTemplate"
	          p:connectionFactory-ref="jedisConnFactory"/>

	注意： 里面的 jedisConnFactory链接信息 修改为自己的连接串，这里选择database 1为存放cas票据的数据库


* 重新编译 mvn install 

		 生成的cas.war 即可以部署到多个容器中实现负载均衡了，最好配置下容器的session共享
		 参考： https://github.com/izerui/tomcat-redis-session-manager
