<div align="center">
  <h1>CityHub - AI 赋能的本地生活服务平台</h1>
  <p>
    <img src="https://img.shields.io/badge/Spring%20Boot-3.0-green" alt="Spring Boot">
    <img src="https://img.shields.io/badge/RocketMQ-5.0-blue" alt="RocketMQ">
    <img src="https://img.shields.io/badge/Redis-7.0-red" alt="Redis">
    <img src="https://img.shields.io/badge/Caffeine-3.1-orange" alt="Caffeine">
    <img src="https://img.shields.io/badge/LangChain4j-0.3-purple" alt="LangChain4j">
    <img src="https://img.shields.io/badge/License-MIT-yellow" alt="License">
  </p>
</div>

<br/>

是一个融合AI大模型的高并发本地生活服务平台。用户可以浏览商家信息、发布探店笔记、参与优惠券秒杀，并体验基于AI的智能导购与客服功能。

重点攻克了“高并发秒杀”、“热点数据查询”及“分布式数据一致性”等技术难题。针对传统电商系统中存在的超卖、缓存击穿、数据库宕机等痛点，设计了多级缓存架构与全链路异步削峰方案。

[本项目拉取了某著名点评项目用于学习参考]

##  核心功能



- 设计 **Redis + Lua** 实现高并发库存扣减与一人一单校验，解决分布式场景下的超卖问题，保障数据强一致性。

- 使用逻辑过期方案防止 Redis 热点 Key 的**缓存击穿**问题,使用缓存空值方案解决 Redis Key 的**缓存穿透**问题。

- 构建 **Caffeine + Redis 二级缓存架构**，将热点商户数据存储在本地堆缓存，大幅降低 Redis 网络IO开销，接口QPS提升明显。

- 使用 **Redis + AOP+注解**实现限流，支持全局、IP、用户多维度，防止系统过载、刷券、爬虫。

- 基于 **RocketMQ **实现流量削峰填谷，将同步下单流程优化为异步消息处理，显著提升秒杀瞬间吞吐量。

- 使用**乐观锁**解决订单支付与超时关单时的并发冲突，利用**Spring Task**实现未支付订单的最终一致性兜底处理。

- 接入硅基流动平台大模型，使用Redis支持会话记忆，基于 Function Calling 实现查询信息和预约到店。

<br/>

## 技术栈

### 后端

SpringBoot 3 + MyBatis Plus + RocketMQ + Redis + Caffeine + LangChain4j + MySQL

### 部署

Docker



<br/>

## 我的开发环境 

| 组件 | 版本 | 备注 |
| :--- | :--- | :--- |
| **JDK** | 17 / 21 | 推荐使用 JDK 17 LTS |
| **MySQL** | 8.0 | Docker 镜像 `mysql:8.0` |
| **Redis** | 7.0 | Docker 镜像 `redis:7.0` |
| **RocketMQ** | 5.1.0 | 包含 NameServer 与 Broker |
| **Nginx** | Latest | 用于负载均衡与动静分离 |

<br/>

# 核心逻辑

> 所有技术选型都是为了解决“高并发下的快”和“分布式下的稳”。

---

## 1. 用户登录 & 会话共享
**场景**：集群环境下，Tomcat A 存的 Session，Tomcat B 拿不到。
**解决方案**：**Redis + Token**
* **原理**：
    * 生成随机 Token 作为 Key，用户信息作为 Value 存入 Redis（设置 TTL）。
    * 将 Token 返回给前端，前端每次请求在 Header 携带 Token。
* **拦截器设计（双层）**：
    1.  **全局拦截器**：拦截所有请求，刷新 Token 有效期（保证活跃用户不掉线）。
    2.  **登录拦截器**：拦截部分路径，判断 ThreadLocal 是否有用户，无则拦截。
* **为什么**：
    * *Q: 为什么不用 Session？*
    * A: Session 存在单机内存，集群无法共享。
    * *Q: 为什么用两层拦截器？*
    * A: 为了“只要用户在操作就不断签”。第一层负责刷新 Redis TTL，第二层负责鉴权。

---

## 2. 缓存三兄弟 (穿透/击穿/雪崩)
**核心思想**：保护数据库。

| 问题 | 定义 | 解决方案 (本项目) |
| :--- | :--- | :--- |
| **缓存穿透** | 查**不存在**的数据，请求直打数据库 | **缓存空对象** (设置短 TTL) + **布隆过滤器** (可选) |
| **缓存雪崩** | 大量 Key **同时过期**，DB 压力剧增 | **随机 TTL** (原时间 + 随机值) |
| **缓存击穿** | **热点 Key** 过期，并发请求击穿 DB | **逻辑过期** (互斥锁方案虽然强一致但性能低，逻辑过期适合高并发) |

**逻辑过期实现细节**：
1.  Redis 存的数据里包含一个“逻辑过期时间”字段，物理上永不过期。
2.  查询时判断逻辑时间已过期 -> 获取互斥锁 -> **开启独立线程**去重构缓存 -> 主线程直接返回旧数据（牺牲强一致性换取高可用）。

---

## 3. 秒杀核心 (高并发 + 超卖)
**场景**：1000 人抢 10 张券。

### 3.1 解决超卖 (乐观锁)
* **痛点**：线程 A 查询库存为 1，线程 B 也查为 1，都扣减成功，库存变 -1。
* **方案**：CAS (Compare And Swap)。
* **SQL**：`UPDATE voucher SET stock = stock - 1 WHERE id = ? AND stock > 0`
* **面试题**：*为什么不用 `stock = oldVersion`？* -> 因为只要库存 > 0 就能卖，太严格会导致成功率极低。

### 3.2 一人一单 (分布式锁)
* **痛点**：单机锁 (`synchronized`) 在集群模式下失效（不同 JVM 锁不住）。
* **方案**：**Redisson 分布式锁**。
* **锁 Key 设计**：`lock:order:{userId}` (锁用户，而不是锁整个方法，提升性能)。
* **Redisson 核心原理 (看门狗 WatchDog)**：
    * 默认 30s 超时。
    * 业务没跑完，每隔 10s 自动续期，防止业务没做完锁就断了。
    * **可重入原理**：利用 Hash 结构记录线程 ID 和重入次数。

### 3.3 极致性能优化 (Redis + Lua + MQ)
**旧流程**：查库 -> 判重 -> 扣库 -> 创单 (全数据库操作，慢)。
**新流程 (异步架构)**：
1.  **Redis 预减**：使用 **Lua 脚本** 原子性判断库存和校验一人一单。
    * *Lua 作用*：保证 3 个 Redis 操作（查库、判断用户 Set、扣减）原子执行，不会被插队。
2.  **MQ 削峰**：Lua 判断成功后，发送消息到 RocketMQ/RabbitMQ，立即返回前端“排队中”。
3.  **DB 兜底**：消费者慢慢消费消息，写数据库。

---

## 4. 消息队列 (RabbitMQ/RocketMQ)
* **作用**：解耦、削峰、异步。
* **死信队列 (DLQ)**：
    * 订单超时未支付 -> 进入死信队列 -> 消费者取消订单、回滚库存。
* **面试题**：*如何保证消息不丢失？*
    * 生产者 Confirm 机制。
    * 消费者手动 ACK。
    * 队列持久化。

---

## 5. 特殊场景数据结构

### 5.1 点赞排行榜 (Redis ZSet)
* **需求**：按时间排序的点赞列表。
* **实现**：`ZADD key score(时间戳) value(userId)`。
* **判断点赞**：`ZSCORE key userId`，有值就是点过。

### 5.2 共同关注 (Redis Set)
* **需求**：A 关注的人和 B 关注的人取交集。
* **实现**：`SINTER keyA keyB` (求交集)。

### 5.3 附近商户 (Redis GEO)
* **需求**：搜索方圆 5km 内的店，按距离排序。
* **实现**：底层是 GeoHash 算法（将二维坐标转为一维字符串）。
* **命令**：`GEORADIUS` 或 `GEOSEARCH`。

### 5.4 用户签到 (Redis BitMap)
* **需求**：记录用户一年的签到，极其节省空间。
* **原理**：1 个 bit 代表一天，0 未签，1 已签。
* **统计**：
    * `SETBIT key offset 1` (签到)。
    * `BITCOUNT` (统计总天数)。
    * **连续签到计算**：取出本月所有 bit，通过位运算（右移 + 与运算）判断末尾有多少个连续的 1。

### 5.5 UV 统计 (HyperLogLog)
* **需求**：统计网站即使百万级访问量，误差允许极小。
* **原理**：概率算法，占用内存极小（12KB），标准误差 0.81%。
* **命令**：`PFADD` (添加), `PFCOUNT` (统计)。



