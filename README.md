# 实现cas ticket基于redis的集群

### 目的
	克服cas单点故障，将cas认证请求分发到多台cas服务器上，降低负载。

### 实现思路：
	采用统一的ticket存取策略，所有ticket的操作都从中央缓存redis中存取。
	采用session共享，session的存取都从中央缓存redis中存取。

### 前提：
	这里只讲解如何实现cas ticket的共享，关于session的共享请移步：




- <a href="https://github.com/izerui/tomcat-redis-session-manager">https://github.com/izerui/tomcat-redis-session-manager</a>

### 实现步骤：



1. 添加依赖到  **cas-server-core**模块的 pom.xml


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



2. 添加类到 **cas-server-core**模块的 org.jasig.cas.ticket.registry 包下

	**RedisTicketRegistry.java**

		package org.jasig.cas.ticket.registry;

		import org.jasig.cas.ticket.Ticket;
		import org.springframework.util.Assert;
		
		import java.util.Collection;
		import java.util.HashSet;
		import java.util.Set;
		
		/**
		 * Created by serv on 2015/7/18.
		 */
		public final class RedisTicketRegistry extends AbstractTicketRegistry{
		
		    private final static String TICKET_PREFIX = "TICKETGRANTINGTICKET:";
		
		    private TicketRedisTemplate redisTemplate;
		
		    public void setRedisTemplate(TicketRedisTemplate redisTemplate) {
		        this.redisTemplate = redisTemplate;
		    }
		
		    @Override
		    public void addTicket(Ticket ticket) {
		        Assert.notNull(ticket, "ticket cannot be null");
		        logger.debug("Added ticket [{}] to registry.", ticket.getId());
		
		        redisTemplate.boundValueOps(TICKET_PREFIX+ticket.getId()).set(ticket);
		    }
		
		    @Override
		    public Ticket getTicket(String ticketId) {
		        if (ticketId == null) {
		            return null;
		        }
		
		        logger.debug("Attempting to retrieve ticket [{}]", ticketId);
		
		        Ticket ticket = redisTemplate.boundValueOps(TICKET_PREFIX + ticketId).get();
		
		        if (ticket != null) {
		            logger.debug("Ticket [{}] found in registry.", ticketId);
		        }
		        return ticket;
		    }
		
		    @Override
		    public boolean deleteTicket(String ticketId) {
		        if (ticketId == null) {
		            return false;
		        }
		        logger.debug("Removing ticket [{}] from registry", ticketId);
		        redisTemplate.delete(TICKET_PREFIX+ticketId);
		        return true;
		    }
		
		    @Override
		    public Collection<Ticket> getTickets() {
		        Set<Ticket> tickets = new HashSet<Ticket>();
		        Set<String> keys = redisTemplate.keys(TICKET_PREFIX + "*");
		        for (String key:keys){
		            tickets.add(redisTemplate.boundValueOps(TICKET_PREFIX+key).get());
		        }
		        return tickets;
		    }
		
		}

	**TicketRedisTemplate.java**

		package org.jasig.cas.ticket.registry;

		import org.jasig.cas.ticket.Ticket;
		import org.springframework.data.redis.connection.RedisConnectionFactory;
		import org.springframework.data.redis.core.RedisTemplate;
		import org.springframework.data.redis.serializer.JdkSerializationRedisSerializer;
		import org.springframework.data.redis.serializer.RedisSerializer;
		import org.springframework.data.redis.serializer.StringRedisSerializer;
		
		/**
		 * Created by serv on 2015/7/19.
		 */
		public class TicketRedisTemplate extends RedisTemplate<String, Ticket> {
		
		    public TicketRedisTemplate() {
		        RedisSerializer<String> string = new StringRedisSerializer();
		        JdkSerializationRedisSerializer jdk = new JdkSerializationRedisSerializer();
		        setKeySerializer(string);
		        setValueSerializer(jdk);
		        setHashKeySerializer(string);
		        setHashValueSerializer(jdk);
		    }
		
		    public TicketRedisTemplate(RedisConnectionFactory connectionFactory) {
		        this();
		        setConnectionFactory(connectionFactory);
		        afterPropertiesSet();
		    }
		}





3. 修改 spring配置文件 **cas-server-webapp** 模块 WEB-INF\spring-configuration\ticketRegistry.xml

		
		<bean id="ticketRegistry" class="org.jasig.cas.ticket.registry.DefaultTicketRegistry" />

	替换为

		<bean id="ticketRegistry" class="org.jasig.cas.ticket.registry.RedisTicketRegistry"
          p:redisTemplate-ref="redisTemplate"/>

	    <bean id="jedisConnFactory"	class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
	          p:hostName="182.92.4.161"
	          p:database="1"
	          p:usePool="true"/>
	
	    <bean id="redisTemplate" class="org.jasig.cas.ticket.registry.TicketRedisTemplate"
	          p:connectionFactory-ref="jedisConnFactory"/>

	注意： 里面的 jedisConnFactory链接信息 修改为自己的连接串，这里选择database 1为存放cas票据的数据库




4. 重新编译 mvn install 

		 生成的cas.war 部署到多个已经做过session共享的tomcat容器中。
