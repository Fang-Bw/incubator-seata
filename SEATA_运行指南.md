# Seata 项目运行指南

## 项目简介

Apache Seata 是一个开源的分布式事务解决方案，致力于在微服务架构下提供高性能和简单易用的分布式事务服务。

## 系统要求

### 必需环境
- **Java 版本**: JDK 1.8 或更高版本
- **构建工具**: Maven 3.6+ （项目自带 Maven Wrapper）
- **操作系统**: Linux、macOS、Windows

### 推荐配置
- **内存**: 至少 2GB 可用内存
- **磁盘空间**: 至少 1GB 可用空间
- **网络**: 确保端口 8091 可用

## 快速开始

### 1. 项目编译

#### 基础编译
```bash
# 进入项目根目录
cd /path/to/incubator-seata

# 使用 Maven Wrapper 编译
./mvnw clean compile -DskipTests
```

#### 完整编译（推荐）
```bash
# 编译整个项目，跳过测试
./mvnw clean install -DskipTests -Prelease-seata
```

**编译选项说明：**
- `-DskipTests`: 跳过测试执行
- `-Prelease-seata`: 使用发布配置 profile
- `-U`: 强制更新依赖（可选）

### 2. 运行 Seata Server

#### 方法一：直接运行（推荐）

1. **编译项目**
```bash
./mvnw clean install -DskipTests -Prelease-seata
```

2. **进入 server 目录**
```bash
cd server
```

3. **启动服务**
```bash
java -cp "target/classes:target/lib/*" org.apache.seata.server.ServerApplication
```

#### 方法二：使用 Maven 插件运行

```bash
# 在项目根目录执行
./mvnw spring-boot:run -pl server -Prelease-seata
```

**注意**: 此方法可能遇到类路径问题，推荐使用方法一。

#### 方法三：使用发布包运行

1. **生成发布包**
```bash
./mvnw -Prelease-seata -Dmaven.test.skip=true clean install -U
```

2. **进入发布目录**
```bash
cd distribution/target/apache-seata-*-bin/seata-server/
```

3. **启动服务**
```bash
# Linux/macOS
./bin/seata-server.sh

# Windows
bin\seata-server.bat
```

### 3. Docker 运行方式

#### 构建 Docker 镜像

1. **编译项目**
```bash
./mvnw -Prelease-seata -Dmaven.test.skip=true clean install -U
```

2. **进入 server 目录**
```bash
cd distribution/target/apache-seata-*-bin/seata-server/
```

3. **构建镜像**
```bash
docker build --no-cache --build-arg SEATA_VERSION=2.6.0-SNAPSHOT -t seata-server:dev .
```

#### 运行容器

```bash
docker run --name=seata-server -p 8091:8091 -d seata-server:dev
```

## 验证运行状态

### 检查服务状态

1. **健康检查**
```bash
curl http://localhost:8091/health
```
预期返回: `"ok"`

2. **检查端口占用**
```bash
lsof -i :8091
# 或
netstat -an | grep 8091
```

3. **查看日志**
```bash
# 默认日志路径
tail -f /Users/$(whoami)/logs/seata/seata-server.log
```

### 启动成功标志

当看到以下日志时，表示服务启动成功：

```
INFO --- Server started, service listen port: 8091
INFO --- you can visit seata console UI on namingserver.
INFO --- seata server started in XXX millSeconds
```

## 配置说明

### 主要配置文件

- **application.yml**: 主配置文件
- **registry.conf**: 注册中心配置
- **file.conf**: 文件存储配置（默认模式）

### 配置文件位置

```
server/src/main/resources/
├── application.yml
├── registry.conf
├── file.conf
└── logback-spring.xml
```

### 常用配置项

#### application.yml 示例配置
```yaml
server:
  port: 7091
seata:
  config:
    type: file
  registry:
    type: file
  store:
    mode: file
```

## 常见问题解决

### 1. 编译问题

**问题**: Maven 依赖下载失败
```bash
# 解决方案：清理本地仓库并重新下载
rm -rf ~/.m2/repository/org/apache/seata
./mvnw clean install -U
```

### 2. 运行问题

**问题**: `NoClassDefFoundError: SpringApplication`
```bash
# 解决方案：确保使用正确的类路径
java -cp "target/classes:$(find target/lib -name '*.jar' | tr '\n' ':')" org.apache.seata.server.ServerApplication
```

**问题**: 端口 8091 被占用
```bash
# 查找占用进程
lsof -i :8091
# 终止进程
kill -9 <PID>
```

### 3. 事务回滚问题排查

**问题现象：**
- 全局事务状态显示为 `Rollbacked`
- 但数据库中的数据没有实际回滚
- 日志中缺少分支事务注册信息

**关键日志特征：**
```
GlobalTransactionContext.reload(xid) - 全局事务回滚成功
但缺少：Branch transaction xxx registered successfully
```

**解决方案：**

#### 1. **检查 @GlobalTransactional 注解配置**
```java
// ❌ 错误配置 - 可能不会回滚某些异常
@GlobalTransactional(name = "business-transaction")
public void businessMethod() {
    // 检查异常默认不会回滚
    throw new SQLException("数据库异常");
}

// ✅ 正确配置 - 明确指定回滚异常
@GlobalTransactional(
    name = "business-transaction",
    rollbackFor = Exception.class,  // 所有异常都回滚
    timeoutMills = 30000
)
public void businessMethod() {
    // 业务逻辑
}

// ✅ 或者指定具体异常类型
@GlobalTransactional(
    rollbackFor = {SQLException.class, BusinessException.class}
)
public void businessMethod() {
    // 业务逻辑
}
```

#### 2. **XA 事务模式特殊配置**
如果使用 XA 模式，需要额外配置：

**Seata Server 配置 (application.yml):**
```yaml
seata:
  server:
    # XA 事务相关配置
    xaer-nota-retry-timeout: 60000
    enable-check-auth: true
    recovery:
      committing-retry-period: 1000
      async-committing-retry-period: 1000
      rollbacking-retry-period: 1000
      timeout-retry-period: 1000
```

**业务应用数据源配置:**
```yaml
spring:
  datasource:
    # 使用支持 XA 的数据源
    type: com.atomikos.jdbc.AtomikosDataSourceBean
    # 或者配置 XA 数据源属性
    xa:
      data-source-class-name: com.mysql.cj.jdbc.MysqlXADataSource
      properties:
        url: jdbc:mysql://localhost:3306/seata
        user: root
        password: password
```

#### 3. **检查数据源代理配置**
   - 确保数据源被 Seata 正确代理
   - 检查 `@EnableAutoDataSourceProxy` 注解
   - 验证数据源配置是否正确

#### 4. **验证事务管理器配置**
   - 确认 Seata 客户端配置正确
   - 检查事务组配置是否匹配

#### 5. **XA数据源配置要求**

**重要：使用XA事务模式时，业务应用必须配置XA数据源，不能使用普通JDBC数据源！**

##### 错误配置（普通JDBC数据源）：
```properties
# ❌ 这种配置无法支持XA事务
spring.datasource.driverClassName=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/seata?useSSL=false
spring.datasource.username=root
spring.datasource.password=123456
```

##### 正确配置（XA数据源）：

**方案1：使用Atomikos（推荐）**
```properties
# 应用配置
spring.application.name=springboot-feign-seata-business
server.port=8084

# XA数据源配置
spring.jta.atomikos.datasource.primary.xa-data-source-class-name=com.mysql.cj.jdbc.MysqlXADataSource
spring.jta.atomikos.datasource.primary.xa-properties.url=jdbc:mysql://127.0.0.1:3306/seata?useSSL=false&useUnicode=true&characterEncoding=UTF8
spring.jta.atomikos.datasource.primary.xa-properties.user=root
spring.jta.atomikos.datasource.primary.xa-properties.password=123456
spring.jta.atomikos.datasource.primary.unique-resource-name=primary
spring.jta.atomikos.datasource.primary.max-pool-size=10
spring.jta.atomikos.datasource.primary.min-pool-size=1

# Seata配置
seata.application-id=springboot-feign-seata-business
seata.tx-service-group=my_test_tx_group
seata.mode=xa
```

**方案2：Java配置方式**
```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary
    public DataSource dataSource() {
        AtomikosDataSourceBean dataSource = new AtomikosDataSourceBean();
        dataSource.setUniqueResourceName("primary");
        dataSource.setXaDataSourceClassName("com.mysql.cj.jdbc.MysqlXADataSource");
        
        Properties properties = new Properties();
        properties.setProperty("url", "jdbc:mysql://127.0.0.1:3306/seata?useSSL=false");
        properties.setProperty("user", "root");
        properties.setProperty("password", "123456");
        dataSource.setXaProperties(properties);
        
        dataSource.setMaxPoolSize(10);
        dataSource.setMinPoolSize(1);
        
        return dataSource;
    }
    
    @Bean
    public JtaTransactionManager transactionManager() {
        UserTransactionManager userTransactionManager = new UserTransactionManager();
        UserTransaction userTransaction = new UserTransactionImp();
        return new JtaTransactionManager(userTransaction, userTransactionManager);
    }
}
```

**必需依赖：**
```xml
<dependency>
    <groupId>com.atomikos</groupId>
    <artifactId>transactions-spring-boot-starter</artifactId>
    <version>4.0.6</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

#### 6. **异常处理最佳实践**
```java
@GlobalTransactional(rollbackFor = Exception.class)
public void businessMethod() {
    try {
        // 业务逻辑
        updateDatabase();
    } catch (Exception e) {
        // 记录日志
        log.error("业务执行失败", e);
        // 重新抛出异常，确保事务回滚
        throw e;
    }
}
```

### 4. 性能优化

**JVM 参数建议**:
```bash
java -Xms2g -Xmx2g -XX:+UseG1GC \
     -cp "target/classes:target/lib/*" \
     org.apache.seata.server.ServerApplication
```

## 开发环境配置

### IDE 运行配置

1. **主类**: `org.apache.seata.server.ServerApplication`
2. **工作目录**: `server` 模块目录
3. **类路径**: 包含 `target/classes` 和所有依赖 jar
4. **JVM 参数**: `-Dspring.profiles.active=dev`

### 调试模式

```bash
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 \
     -cp "target/classes:target/lib/*" \
     org.apache.seata.server.ServerApplication
```

## 生产环境部署

### 系统服务配置

创建 systemd 服务文件 `/etc/systemd/system/seata.service`:

```ini
[Unit]
Description=Seata Server
After=network.target

[Service]
Type=simple
User=seata
WorkingDirectory=/opt/seata
ExecStart=/usr/bin/java -jar seata-server.jar
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### 监控和日志

- **日志路径**: `/var/log/seata/`
- **监控端点**: `http://localhost:8091/actuator/health`
- **JMX 端口**: 可通过 JVM 参数配置

## 相关链接

- **官方文档**: https://seata.apache.org/
- **GitHub 仓库**: https://github.com/apache/incubator-seata
- **问题反馈**: https://github.com/apache/incubator-seata/issues

## 版本信息

- **当前版本**: 2.6.0-SNAPSHOT
- **Spring Boot 版本**: 2.7.18
- **Java 版本要求**: 1.8+

---

*最后更新时间: 2025-09-17*