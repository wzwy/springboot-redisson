# springboot-redisson


package com.wz.springboot.configure;

import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RedissonManager {

    @Value("${redisson.address}")
    private String addressUrl;

    @Bean
    public RedissonClient getRedisson() throws Exception {
        RedissonClient redisson = null;
        Config config = new Config();
        config.useSingleServer().setAddress(addressUrl);//指定ip和端口
        redisson = Redisson.create(config);//生成RedissonClient客户端
        System.out.println(redisson.getConfig().toJSON().toString());
        return redisson;
    }
}

package com.wz.springboot.controller;

import com.wz.springboot.service.DistributedLocker;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.concurrent.TimeUnit;

@RestController
@RequestMapping("/redisson")
public class LockTestController {
    @Autowired
    private DistributedLocker distributedLocker;

    @RequestMapping("/test")
    public void redissonTest() {
        String key = "redisson_key";
        for (int i = 0; i < 100; i++) {
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        /*
                         * 直接加锁，获取不到锁则一直等待获取锁
                         * distributedLocker.lock(key,10L);
                         * 获得锁之后可以进行相应的处理
                         * Thread.sleep(100);
                         * System.err.println("======获得锁后进行相应的操作======"+Thread.currentThread().getName());
                         * 解锁
                         * distributedLocker.unlock(key);
                         */
                        // 尝试获取锁，等待5秒，自己获得锁后一直不解锁则10秒后自动解锁
                        boolean isGetLock = distributedLocker.tryLock(key, TimeUnit.SECONDS, 5L, 10L);
                        if (isGetLock) {
                            // 获得锁之后可以进行相应的处理
                            System.out.println("线程:" + Thread.currentThread().getName() + ",获取到了锁");
                            Thread.sleep(100);
                            System.err.println("======获得锁后进行相应的操作======" + Thread.currentThread().getName());
                            // 释放锁
                            distributedLocker.unlock(key);
                            System.err.println("================释放锁=============" + Thread.currentThread().getName());
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            });
            t.start();
        }
    }
}

package com.wz.springboot.service;

import org.redisson.api.RLock;

import java.util.concurrent.TimeUnit;

public interface DistributedLocker {

    public RLock lock(String lockKey);

    public RLock lock(String lockKey, long timeout);

    public RLock lock(String lockKey, TimeUnit unit, long timeout);

    public boolean tryLock(String lockKey, TimeUnit unit, long waitTime, long leaseTime);

    public void unlock(String lockKey);

    public void unlock(RLock lock);
}


package com.wz.springboot.service;

import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class RedissonDistributedLocker implements DistributedLocker{

    // RedissonClient已经由配置类生成，这里自动装配即可
    @Autowired
    private RedissonClient redissonClient;

    // lock(), 拿不到lock就不罢休，不然线程就一直block
    @Override
    public RLock lock(String lockKey) {
        RLock lock = redissonClient.getLock(lockKey);
        lock.lock();
        return lock;
    }

    // leaseTime为加锁时间，单位为秒
    @Override
    public RLock lock(String lockKey, long leaseTime) {
        RLock lock = redissonClient.getLock(lockKey);
        lock.lock(leaseTime, TimeUnit.SECONDS);
        return null;
    }

    // timeout为加锁时间，时间单位由unit确定
    @Override
    public RLock lock(String lockKey, TimeUnit unit, long timeout) {
        RLock lock = redissonClient.getLock(lockKey);
        lock.lock(timeout, unit);
        return lock;
    }

    // 尝试获取锁，等待waitTime秒，自己获得锁后一直不解锁则leaseTime秒后自动解锁，unit单位
    @Override
    public boolean tryLock(String lockKey, TimeUnit unit, long waitTime, long leaseTime) {
        RLock lock = redissonClient.getLock(lockKey);
        try {
            return lock.tryLock(waitTime, leaseTime, unit);
        } catch (InterruptedException e) {
            return false;
        }
    }

    //释放锁
    @Override
    public void unlock(String lockKey) {
        RLock lock = redissonClient.getLock(lockKey);
        lock.unlock();
    }

    //指定RLock对象释放锁
    @Override
    public void unlock(RLock lock) {
        lock.unlock();
    }

}

package com.wz.springboot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication//springboot主启动类
public class SpringbootApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringbootApplication.class, args);
	}

}


<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

	<!--springboot最终打的jar包-->
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.wz</groupId>
	<artifactId>springboot-redisson</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>springboot-redisson</name>
	<description>Spring Boot</description>

	<!--springboot父jar包-->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.0.5.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<!--jar版本-->
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<!--springmvc-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<!--lombok-->
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<version>1.18.2</version>
			<optional>true</optional>
		</dependency>

		<!--redisson-->
		<dependency>
			<groupId>org.redisson</groupId>
			<artifactId>redisson</artifactId>
			<version>3.5.0</version>
		</dependency>

	</dependencies>

</project>



