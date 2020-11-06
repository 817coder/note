# 笔记

### Linux安装Docker
```
1. https://docs.docker.com/官网docs获取各个系统docker的安装教程，按照步骤安装就好了

2. 安装完成： docker -v   查看安装是否成功

3. systemctl start docker    启动docker

4. systemctl enable docker    设置docker自启动

5. 配置镜像加速器
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://v2ltjwbg.mirror.aliyuncs.com"]
}
EOF
systemctl daemon-reload
systemctl restart docker
```

### docker中安装mysql
```
[root@hadoop-104 module]# docker pull mysql:5.7
5.7: Pulling from library/mysql
123275d6e508: Already exists 
27cddf5c7140: Pull complete 
c17d442e14c9: Pull complete 
2eb72ffed068: Pull complete 
d4aa125eb616: Pull complete 
52560afb169c: Pull complete 
68190f37a1d2: Pull complete 
3fd1dc6e2990: Pull complete 
85a79b83df29: Pull complete 
35e0b437fe88: Pull complete 
992f6a10268c: Pull complete 
Digest: sha256:82b72085b2fcff073a6616b84c7c3bcbb36e2d13af838cec11a9ed1d0b183f5e
Status: Downloaded newer image for mysql:5.7
docker.io/library/mysql:5.7
```

启动MySQL
```
sudo docker run -p 3306:3306 --name mysql \
-v /mydata/mysql/log:/var/log/mysql \
-v /mydata/mysql/data:/var/lib/mysql \
-v /mydata/mysql/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7
```

修改配置
```
[root@hadoop-104 conf]# pwd
/mydata/mysql/conf


[root@hadoop-104 conf]# cat my.cnf
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
[mysqld]
init_connect='SET collation_connection = utf8_unicode_ci'
init_connect='SET NAMES utf8'
character-set-server=utf8
collation-server=utf8_unicode_ci
skip-character-set-client-handshake
skip-name-resolve
[root@hadoop-104 conf]# 

[root@hadoop-104 conf]# docker restart mysql
mysql
[root@hadoop-104 conf]# 

``` 

进入容器查看配置：

```
[root@hadoop-104 conf]# docker exec -it mysql /bin/bash
root@b3a74e031bd7:/# whereis mysql
mysql: /usr/bin/mysql /usr/lib/mysql /etc/mysql /usr/share/mysql

root@b3a74e031bd7:/# ls /etc/mysql 
my.cnf
root@b3a74e031bd7:/# cat /etc/mysql/my.cnf 
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
[mysqld]
init_connect='SET collation_connection = utf8_unicode_ci'
init_connect='SET NAMES utf8'
character-set-server=utf8
collation-server=utf8_unicode_ci
skip-character-set-client-handshake
skip-name-resolve
root@b3a74e031bd7:/# 
```

设置启动docker时，即运行mysql

```
[root@hadoop-104 ~]# docker update mysql --restart=always
mysql
[root@hadoop-104 ~]# 
```

### docker中安装redis

下载redis镜像
```
[root@hadoop-104 ~]# docker pull redis
Using default tag: latest
latest: Pulling from library/redis
123275d6e508: Already exists 
f2edbd6a658e: Pull complete 
66960bede47c: Pull complete 
79dc0b596c90: Pull complete 
de36df38e0b6: Pull complete 
602cd484ff92: Pull complete 
Digest: sha256:1d0b903e3770c2c3c79961b73a53e963f4fd4b2674c2c4911472e8a054cb5728
Status: Downloaded newer image for redis:latest
docker.io/library/redis:latest
```

启动docker

```
[root@hadoop-104 ~]# mkdir -p /mydata/redis/conf
[root@hadoop-104 ~]# touch /mydata/redis/conf/redis.conf
[root@hadoop-104 ~]# echo "appendonly yes"  >> /mydata/redis/conf/redis.conf
[root@hadoop-104 ~]# docker run -p 6379:6379 --name redis -v /mydata/redis/data:/data \
> -v /mydata/redis/conf/redis.conf:/etc/redis/redis.conf \
> -d redis redis-server /etc/redis/redis.conf
ce7ae709711986e3f90c9278b284fe6f51f1c1102ba05f3692f0e934ceca1565
[root@hadoop-104 ~]# 
```

连接到docker的redis

```
[root@hadoop-104 ~]# docker exec -it redis redis-cli
127.0.0.1:6379> set key1 v1
OK
127.0.0.1:6379> get key1
"v1"
127.0.0.1:6379> 
```

设置redis容器在docker启动的时候启动

```
[root@hadoop-104 ~]# docker update redis --restart=always
redis
[root@hadoop-104 ~]# 
```

### maven设置
设置镜像和JDK版本

### github创建项目
新建gulimall项目，分模块构建

### clone 并运行 人人开源

```
git clone https://gitee.com/renrenio/renren-fast-vue.git
git clone https://gitee.com/renrenio/renren-fast.git
```

将拷贝下来的“renren-fast”删除“.git”后，拷贝到“gulimall”工程根目录下，然后将它作为gulimall的一个module

创建“gulimall_admin”的数据库，然后执行“renren-fast/db/mysql.sql”中的SQl脚本

修改“application-dev.yml”文件，默认为dev环境，修改连接mysql的url和用户名密码

```
spring:
    datasource:
        type: com.alibaba.druid.pool.DruidDataSource
        druid:
            driver-class-name: com.mysql.cj.jdbc.Driver
            url: jdbc:mysql://192.168.137.14:3306/gulimall_admin?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai
            username: root
            password: root
```

启动 enrenfast “gulimall_admin”，然后访问 http://localhost:8080/renren-fast/ 查看后端服务是否正常运行

从官网下载并安装node.js，并且安装仓库

```
npm config set registry http://registry.npm.taobao.org/
```
```
PS D:\tmp\renren-fast-vue> npm config set registry http://registry.npm.taobao.org/
PS D:\tmp\renren-fast-vue> npm install
npm WARN ajv-keywords@1.5.1 requires a peer of ajv@>=4.10.0 but none is installed. You must install peer dependencies yourself.
npm WARN sass-loader@6.0.6 requires a peer of node-sass@^4.0.0 but none is installed. You must install peer dependencies yourself.
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@1.2.9 (node_modules\fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@1.2.9: wanted {"os":"darwin","arch":"any"} (current: {"os":"win32","arch":"x64"})

up to date in 17.227s
PS D:\tmp\renren-fast-vue>
```

```
PS D:\tmp\renren-fast-vue> npm run dev

> renren-fast-vue@1.2.2 dev D:\tmp\renren-fast-vue
> webpack-dev-server --inline --progress --config build/webpack.dev.conf.js

 10% building modules 5/10 modules 5 active ...-0!D:\tmp\renren-fast-vue\src\main.js(node:19864) Warning: Accessing non-existent property 'cat' of module exports inside circular dependency
(Use `node --trace-warnings ...` to show where the warning was created)
(node:19864) Warning: Accessing non-existent property 'cd' of module exports inside circular dependency
(node:19864) Warning: Accessing non-existent property 'chmod' of module exports inside circular dependency
(node:19864) Warning: Accessing non-existent property 'cp' of module exports inside circular dependency
(node:19864) Warning: Accessing non-existent property 'dirs' of module exports inside circular dependency
(node:19864) Warning: Accessing non-existent property 'pushd' of module exports inside circular dependency
(node:19864) Warning: Accessing non-existent property 'popd' of module exports inside circular dependency
(node:19864) Warning: Accessing non-existent property 'echo' of module exports inside circular dependency
(node:19864) Warning: Accessing non-existent property 'tempdir' of module exports inside circular dependency
(node:19864) Warning: Accessing non-existent property 'pwd' of module exports inside circular dependency
```

### clone renren-generator

clone
https://gitee.com/renrenio/renren-generator.git

然后将该项目放置到“gulimall”的跟路径下，然后添加该Module，并且提交到github上

修改配置

renren-generator/src/main/resources/generator.properties

#代码生成器，配置信息

```
mainPath=com.wang
#包名
package=com.wang.gulimall
moduleName=product
#作者
author=wang
#Email
email=1916622321@qq.com
#表前缀(类名不会包含表前缀)
tablePrefix=pms_
```

运行“renren-generator”

访问：<http://localhost:80/

然后点击“生成代码”，将下载的“renren.zip”，解压后取出main文件夹，放置到“gulimall-product”项目的main目录中。

下面的几个module，也采用同样的方式来操作。

### 整合mybatis-plus
1）、导入依赖

```
<dependency>

    <groupId>com.baomidou</groupId>

    <artifactId>mybatis-plus-boot-starter</artifactId>

    <version>3.2.0</version>

</dependency>
```

2）、配置

1、配置数据源；

1）、导入数据库的驱动。https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-versions.html

2）、在application.yml配置数据源相关信息

```
spring:
  datasource:
    username: root
    password: root
    url: jdbc:mysql://#:3306/gulimall_pms
    driver-class-name: com.mysql.cj.jdbc.Driver
```

2、配置MyBatis-Plus；

1）、使用@MapperScan

2）、告诉MyBatis-Plus，sql映射文件位置

```
mybatis-plus:
  mapper-locations: classpath:/mapper/**/*.xml
  global-config:
    db-config:
    	#主键自增
      id-type: auto
```

### SpringCloud组件选型

SpringCloud的几大痛点：
- 部分组件停止维护和更新，给开发带来不便
- 部分环境搭建复杂，没有完善的可视化洁面，需要繁琐的二次开发
- 配置复杂，难上手，部分配置差别难以区分和合理使用

SpringCloud Alibaba的优势：
阿里使用过的组件，经过实战考验，性能强悍，设计合理
成套的产品搭配完善的可视化界面，给开发和运维提供了极大的遍历
上手简单，学习曲线低

结合上面的SpringCloud Alibaba最终的技术搭配方案：
- SpringCloud Alibaba - Nacos   注册中心
- SpringCloud Alibaba - Nacos   配置中心
- SpringCloud - Ribon    负载均衡
- SpringCloud - Feign    声明式Http客户端（调用远程服务）
- SpringCloud Alibaba - Sentinel  服务容错
- SpringCloud - Gateway  API网关
- SpringCloud - Sleuth   调用链监控
- SpringCloud Alibaba - Seata   分布式事务

### Nacos安装运行

安装Nacos

首先下载Nacos 的最新版本 https://github.com/alibaba/nacos/releases

```
// 首先官网下载并解压
cd nacos/bin 
sh startup.sh -m standalone     如果报错     bash -f ./startup.sh -m standalone
```

Nacos服务控制台：http://49.235.32.38:8848/nacos/index.html#/login      默认登陆账号密码： nacos nacos

### 微服务注册中心

搭建nacos集群，然后分别启动各个微服务，将它们注册到Nacos中。

1. 首先配置Nacos注册中心配置
```
  application:
    name: gulimall-coupon
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.137.14
```

2. 启动类增加注解  @EnableDiscoveryClient    开启自动注册功能

3. 查看注册情况：http://49.235.32.38:8848/nacos/index.html#/serviceManagement?dataId=&group=&appName=&namespace=


### 使用openfen

1)、引入open-feign
```
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
```

2)、编写一个接口，告诉SpringCLoud这个接口需要调用远程服务

修改“io.niceseason.gulimall.coupon.controller.CouponController”，添加以下controller方法：
```
    @RequestMapping("/member/list")
    public R memberCoupons(){
        CouponEntity couponEntity = new CouponEntity();
        couponEntity.setCouponName("discount 20%");
        return R.ok().put("coupons",Arrays.asList(couponEntity));
    }
```

新建“io.niceseason.gulimall.member.feign.CouponFeignService”接口

```
@FeignClient("gulimall_coupon")
public interface CouponFeignService {
    @RequestMapping("/coupon/coupon/member/list")
    public R memberCoupons();
}
```

为主类添加上"@EnableFeignClients"：声明接口的每一个方法都是调用哪个远程服务的那个请求

```
@EnableFeignClients("com.wang.gulimall.member.feign")
@MapperScan("com.wang.gulimall.member.dao")
@SpringBootApplication
@EnableDiscoveryClient
public class GulimallMemberApplication {

    public static void main(String[] args) {
        SpringApplication.run(GulimallMemberApplication.class, args);
    }
}
```



3)、开启远程调用功能

```
@Autowired
    CouponFeignService couponFeignService;

    @RequestMapping("/coupons")
    public R test(){

        MemberEntity entity = new MemberEntity();
        entity.setNickname("wang");
        R r = couponFeignService.membersCoupons();

        return R.ok().put("members", entity).put("coupons", r.get("coupons"));

    }
```

(4)、访问http://localhost:8000/member/member/coupons 查看服务响应

如果停止“gulimall-coupon”服务，能够看到注册中心显示该服务的健康值为0：

启动“gulimall-coupon”服务，再次访问，又恢复了正常。


















