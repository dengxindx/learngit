## 目录
- 1[springboot整合redis]()
    - jar包更新(jedis)：
    
        ```text
        **原来的**
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        **更新为**
        <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-data-redis</artifactId>          
        </dependency>
        <dependency>
           <groupId>redis.clients</groupId>
           <artifactId>jedis</artifactId>
        </dependency>
        ```
    