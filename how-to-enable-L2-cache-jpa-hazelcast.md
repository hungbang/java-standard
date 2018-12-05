1. Create new hazelcast instance with docker:
- Using hazelcast management center 3.10 , it will be compatibility with `hazelcast 3.10` too.

`docker run -d --name hazelcast-mgmt -p 38080:8080 hazelcast/management-center:3.10`

2. Extending hazelcast base image 3.10 
- Create new `hazelcast.xml` link this : 
```
<?xml version="1.0" encoding="UTF-8"?>
<hazelcast xmlns="http://www.hazelcast.com/schema/config"
		   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		   xsi:schemaLocation="http://www.hazelcast.com/schema/config
           https://hazelcast.com/schema/config/hazelcast-config-3.7.xsd">
	<group>
		<name>skydiving4sell-hazelcast-group</name>
		<password>password</password>
	</group>
	<management-center enabled="true">http://10.11.1.242:38080/hazelcast-mancenter</management-center>
	<!--<instance-name>skydiving4sell-hazelcast-instance</instance-name>-->
	<network>
		<port auto-increment="true" port-count="100">5701</port>
		<outbound-ports>
			<!-- Allowed port range when connecting to other members. 0 or * means
				the port provided by the system. -->
			<ports>0</ports>
		</outbound-ports>
		<join>
			<multicast enabled="false"></multicast>
			<tcp-ip enabled="false">
				<member-list>
					<member>127.0.0.1</member>
					<member>127.0.0.1</member>
					<member>localhost</member>
				</member-list>
			</tcp-ip>
			<aws enabled="false"></aws>
		</join>
	</network>
	<!--<replicatedmap name="authorizations"></replicatedmap>-->
</hazelcast>
```
* `	<management-center enabled="true">http://10.11.1.242:38080/hazelcast-mancenter</management-center>` : Hazelcast will connect to `public host` `10.11.1.242`
and `host port` `38080`.
- To get public host in ubuntu we run `hostname -I` 
3.  Create new dockerfile that extending from hazelcast base image, we're using hazelcast base version 3.10

```
FROM hazelcast/hazelcast:3.10

ARG HZ_HOME="/opt/hazelcast"

ADD hazelcast.xml ${HZ_HOME}/hazelcast.xml
ENV JAVA_OPTS -Dhazelcast.config=${HZ_HOME}/hazelcast.xml
```
4.  cd to folder that contain above `Dockerfile` and run :

`docker build -t hazelcast:3.10 .`

5. Cd to root folder and run : 

`docker run -d --name hazelcast -p 5701:5701 hazelcast:3.10`

6.  To start and stop docker container: 
start: `docker start [container id]`
stop: `docker stop [container id]`


# How to enable L2 cache in spring boot : 
##  Mandantory dependencies:
```
    <dependency>
        <groupId>com.hazelcast</groupId>
        <artifactId>hazelcast</artifactId>
    </dependency>
    <dependency>
        <groupId>com.hazelcast</groupId>
        <artifactId>hazelcast-client</artifactId>
    </dependency>
    <dependency>
        <groupId>com.hazelcast</groupId>
        <artifactId>hazelcast-hibernate5</artifactId>
    </dependency>
```
- pease check hazelcast and hibernate compatibility version before config.

1.  Add new these bellow lines when create `LocalContainerEntityManagerFactoryBean` bean:

```
        properties.put(org.hibernate.cfg.AvailableSettings.USE_SECOND_LEVEL_CACHE,  dataSourceProperties.getHibernate().getCache().isUseSecondLevelCache());
        properties.put(AvailableSettings.USE_QUERY_CACHE,  dataSourceProperties.getHibernate().getCache().isUseQueryCache());
        properties.put(AvailableSettings.CACHE_REGION_FACTORY, HazelcastCacheRegionFactory.class);
        properties.put("hibernate.cache.hazelcast.use_native_client", true); // tell jpa hibernate to use hazelcast instance that we've created 
        properties.put("hibernate.cache.hazelcast.native_client_address", "10.11.1.242:5701"); // public host and `hazelcast` port number
        properties.put("hibernate.cache.hazelcast.native_client_group", "skydiving4sell-hazelcast-group"); // find in hazelcast.xml file
        properties.put("hibernate.cache.hazelcast.native_client_password", "password"); // find in hazelcast.xml too.

```
2.  To enable caching for entity :
```
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
@Entity
@Table(name = "PRODUCT")
public class Product {
    private String id;
    private String title;
    private String state;
    private String country;
    private String currency;
    private String description;
    private int price;
    private Boolean damages;
    private String images;
```
3.  We also enable caching for repository method: 

3.1 Adding @EnableCaching to main class
```
@SpringBootApplication
`@EnableCaching`
public class PersonApplication {
 
    public static void main(String[] args) {
        SpringApplication.run(PersonApplication.class, args);
    }
 
```

3.2 Adding @Cacheable to repository method: 

```
public interface PersonRepository extends CrudRepository<Person, Integer> {
 
    @Cacheable("findByPesel")
    public List<Person> findByPesel(String pesel);
 
}
```
3.3 We also declare `HazelCastInstance` and `CacheManager` :

```
    @Bean
    HazelcastInstance hazelcastInstance() {
        ClientConfig config = new ClientConfig();
        config.getGroupConfig().setName("dev").setPassword("dev-pass");
        config.getNetworkConfig().addAddress("192.168.99.100");
        config.setInstanceName("cache-1");
        HazelcastInstance instance = HazelcastClient.newHazelcastClient(config);
        return instance;
    }
 
    @Bean
    CacheManager cacheManager() {
        return new HazelcastCacheManager(hazelcastInstance());
    }
    
   ```

4. Clustering

Topic recommendation : https://piotrminkowski.wordpress.com/2017/05/08/jpa-caching-with-hazelcast-hibernate-and-spring-boot/
Source code: https://github.com/hazelcast/hazelcast












