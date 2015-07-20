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



1. 基于cas源码 新增模块 **cas-server-integration-redis**

	pom.xml 文件如下：
		
		<?xml version="1.0" encoding="UTF-8"?>
		<project xmlns="http://maven.apache.org/POM/4.0.0"
		         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
		    <parent>
		        <artifactId>cas-server</artifactId>
		        <groupId>org.jasig.cas</groupId>
		        <version>4.0.2</version>
		    </parent>
		    <modelVersion>4.0.0</modelVersion>
		
		    <artifactId>cas-server-integration-redis</artifactId>
		
		    <dependencies>
		
		        <dependency>
		            <groupId>org.jasig.cas</groupId>
		            <artifactId>cas-server-core</artifactId>
		            <version>${project.version}</version>
		        </dependency>
		
		        <dependency>
		            <groupId>org.jasig.cas</groupId>
		            <artifactId>cas-server-support-saml</artifactId>
		            <version>${project.version}</version>
		            <scope>provided</scope>
		        </dependency>
		
		
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
		    </dependencies>
		
		</project>


2. 添加类到 **cas-server-integration-redis** 模块的 org.jasig.cas.ticket.registry 包下

	**RedisTicketRegistry.java**

		/*
		 * Licensed to Jasig under one or more contributor license
		 * agreements. See the NOTICE file distributed with this work
		 * for additional information regarding copyright ownership.
		 * Jasig licenses this file to you under the Apache License,
		 * Version 2.0 (the "License"); you may not use this file
		 * except in compliance with the License.  You may obtain a
		 * copy of the License at the following location:
		 *
		 *   http://www.apache.org/licenses/LICENSE-2.0
		 *
		 * Unless required by applicable law or agreed to in writing,
		 * software distributed under the License is distributed on an
		 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
		 * KIND, either express or implied.  See the License for the
		 * specific language governing permissions and limitations
		 * under the License.
		 */
		package org.jasig.cas.ticket.registry;
		
		import org.jasig.cas.ticket.ServiceTicket;
		import org.jasig.cas.ticket.Ticket;
		import org.jasig.cas.ticket.TicketGrantingTicket;
		import org.springframework.beans.factory.DisposableBean;
		
		import javax.validation.constraints.Min;
		import javax.validation.constraints.NotNull;
		import java.util.Collection;
		import java.util.HashSet;
		import java.util.Set;
		import java.util.concurrent.TimeUnit;
		
		/**
		 * Key-value ticket registry implementation that stores tickets in redis keyed on the ticket ID.
		 *
		 * @author Scott Battaglia
		 * @author Marvin S. Addison
		 * @since 3.3
		 */
		public final class RedisTicketRegistry extends AbstractDistributedTicketRegistry implements DisposableBean {
		
		    private final static String TICKET_PREFIX = "TICKETGRANTINGTICKET:";
		
		    /** redis client. */
		    @NotNull
		    private final TicketRedisTemplate client;
		
		    /**
		     * TGT cache entry timeout in seconds.
		     */
		    @Min(0)
		    private final int tgtTimeout;
		
		    /**
		     * ST cache entry timeout in seconds.
		     */
		    @Min(0)
		    private final int stTimeout;
		
		
		    /**
		     * Creates a new instance using the given redis client instance, which is presumably configured via
		     * <code>net.spy.redis.spring.redisClientFactoryBean</code>.
		     *
		     * @param client                      redis client.
		     * @param ticketGrantingTicketTimeOut TGT timeout in seconds.
		     * @param serviceTicketTimeOut        ST timeout in seconds.
		     */
		    public RedisTicketRegistry(final TicketRedisTemplate client, final int ticketGrantingTicketTimeOut,
		                               final int serviceTicketTimeOut) {
		        this.tgtTimeout = ticketGrantingTicketTimeOut;
		        this.stTimeout = serviceTicketTimeOut;
		        this.client = client;
		    }
		
		    protected void updateTicket(final Ticket ticket) {
		        logger.debug("Updating ticket {}", ticket);
		        try {
		            this.client.boundValueOps(TICKET_PREFIX+ticket.getId()).set(ticket,getTimeout(ticket), TimeUnit.SECONDS);
		        } catch (final Exception e) {
		            logger.error("Failed updating {}", ticket, e);
		        }
		    }
		
		    public void addTicket(final Ticket ticket) {
		        logger.debug("Adding ticket {}", ticket);
		        try {
		            this.client.boundValueOps(TICKET_PREFIX+ticket.getId()).set(ticket,getTimeout(ticket),TimeUnit.SECONDS);
		        }catch (final Exception e) {
		            logger.error("Failed adding {}", ticket, e);
		        }
		    }
		
		    public boolean deleteTicket(final String ticketId) {
		        logger.debug("Deleting ticket {}", ticketId);
		        try {
		            this.client.delete(TICKET_PREFIX+ticketId);
		            return true;
		        } catch (final Exception e) {
		            logger.error("Failed deleting {}", ticketId, e);
		        }
		        return false;
		    }
		
		    public Ticket getTicket(final String ticketId) {
		        try {
		            final Ticket t = (Ticket) this.client.boundValueOps(TICKET_PREFIX+ticketId).get();
		            if (t != null) {
		                return getProxiedTicketInstance(t);
		            }
		        } catch (final Exception e) {
		            logger.error("Failed fetching {} ", ticketId, e);
		        }
		        return null;
		    }
		
		    /**
		     * {@inheritDoc}
		     * This operation is not supported.
		     *
		     * @throws UnsupportedOperationException if you try and call this operation.
		     */
		    @Override
		    public Collection<Ticket> getTickets() {
		        Set<Ticket> tickets = new HashSet<Ticket>();
		        Set<String> keys = this.client.keys(TICKET_PREFIX + "*");
		        for (String key:keys){
		            Ticket ticket = this.client.boundValueOps(TICKET_PREFIX+key).get();
		            if(ticket==null){
		                this.client.delete(TICKET_PREFIX+key);
		            }else{
		                tickets.add(ticket);
		            }
		        }
		        return tickets;
		    }
		
		    public void destroy() throws Exception {
		        //do nothing
		    }
		
		    /**
		     * @param sync set to true, if updates to registry are to be synchronized
		     * @deprecated As of version 3.5, this operation has no effect since async writes can cause registry consistency issues.
		     */
		    @Deprecated
		    public void setSynchronizeUpdatesToRegistry(final boolean sync) {}
		
		    @Override
		    protected boolean needsCallback() {
		        return true;
		    }
		
		    private int getTimeout(final Ticket t) {
		        if (t instanceof TicketGrantingTicket) {
		            return this.tgtTimeout;
		        } else if (t instanceof ServiceTicket) {
		            return this.stTimeout;
		        }
		        throw new IllegalArgumentException("Invalid ticket type");
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



3. **cas-server-webapp** 添加刚才新增模块的依赖

		<dependency>
	      <groupId>org.jasig.cas</groupId>
	      <artifactId>cas-server-integration-redis</artifactId>
	      <version>${project.version}</version>
	    </dependency>


3. 修改 spring配置文件 **cas-server-webapp** 模块 WEB-INF\spring-configuration\ticketRegistry.xml

		
		<bean id="ticketRegistry" class="org.jasig.cas.ticket.registry.DefaultTicketRegistry" />

	替换为

		<bean id="ticketRegistry" class="org.jasig.cas.ticket.registry.RedisTicketRegistry">
	        <constructor-arg index="0" ref="redisTemplate" />
	
	        <!-- TGT timeout in seconds -->
	        <constructor-arg index="1" value="1800" />
	
	        <!-- ST timeout in seconds -->
	        <constructor-arg index="2" value="300" />
	    </bean>
	
	    <bean id="jedisConnFactory"
	          class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
	          p:hostName="192.168.1.89"
	          p:database="0"
	          p:usePool="true"/>
	
	    <bean id="redisTemplate" class="org.jasig.cas.ticket.registry.TicketRedisTemplate"
	          p:connectionFactory-ref="jedisConnFactory"/>

	注意： 里面的 jedisConnFactory链接信息 修改为自己的连接串，这里选择database 1为存放cas票据的数据库




4. 重新编译 mvn install 

		 生成的cas.war 部署到多个已经做过session共享的tomcat容器中。
