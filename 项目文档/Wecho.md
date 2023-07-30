# WEcho
![](https://ydschool-online.nosdn.127.net/tiku/f291c1c8d5dccacd2b95ef4fdc68d07cefd0e92562794e5bcdd21a9729b3455d.jpg)

## 介绍
- 一个简单的树洞和社区APP
- 我给这个程序起了两个名字`Whisper(耳语)`、`Echo(回声)`,目前我在想如何把他们两个组合起来。(我也不知道我为啥能想起这俩词...)

## 简单功能
- 在树洞里，你可以完全匿名，没有人知道你是谁、来自哪里或者有什么故事。这是一个安全的空间，为你提供了一个能够真正放下心防、畅所欲言的机会，无需担心被审查或者被评判。所以，请坦诚地倾听自己的内心、表达自己的情感，让自己得到释放和治愈。
- 当你在树洞里时，可以尽情地倾诉今天导致你不开心的事情，我会陪伴在你身边，倾听你的故事和感受，如果你需要的话，请与我分享。
- 当你在树洞发布内容后,和你感同身受的小伙伴可以为你送上一束美丽的鲜花💐
- 你可以尽情的上传今日发生的趣事和照片(当然了不包括一些违法的内容)
- 树洞功能当中我们使用用户自己设定的名称与头像进行发布树洞内容,但是任何人不能查看发布树洞的用户以及不能评论树洞内容,只允许感同身受的人点击动效爱心和送出平安鲜花(我们想让每一位感统深受的用户能够认真的阅读他人今日的心情。)

## UI设计想法

**说一下我打算设计的UI图片**： 主打一个`"孤独倾诉"`主题：使用暗色调或者灰调，加入孤独的人物形象，呈现社交网络不能提供的独特体验，突出这是一个没有干扰的地方，可以安静地倾听自己和他人的心声。

## 遵循Restful接口的规范
> 此接口并不代表最终接口
> 
一个数据对象大致分为以下五个接口

- GET /treeholes : 获取树洞列表
- POST /treeholes : 发布一条树洞
- GET  /treeholes/{tree_id} : 获取单个树洞内容详细
- PUT  /treeholes/{tree_id}: 修改单个树洞内容详细
- DELETE /treeholes/{tree_id}: 删除单个树洞详细
## 基本的后端技术栈
- Go
- MeiliSearch
- Gorm
- Gin
- Minio
## 项目目录
- 还没写呢

## 项目缓存规范
- RedisKey的规范
```shell
project:module:business:uk
项目名    模块名   业务名  唯一标识
```

### 用户信息模块缓存

- 这部分还没设计完成,等待完善吧。

| Key                             | 类型   | 过期时间 | 说明               |
| ------------------------------- | ------ | -------- | ------------------ |
| Wecho:user:login_user:{user_id} | string | 2天      | 存储用户生成的JWT  |
| Wecho:users:cache               | HSET   | 2天      | 用HSET缓存用户信息 |
|                                 |        |          |                    |
|                                 |        |          |                    |

## 数据库设计

### 用户表结构

```sql
CREATE TABLE `users` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '唯一标识一个用户的编号，通常使用自增长整数来表示',
  `username` varchar(50) NOT NULL COMMENT '注册登录用户名',
  `nickname` varchar(50) DEFAULT '该用户未设置nickname' COMMENT '用户在树洞中使用的昵称，可以包含中英文字符和数字',
  `password` varchar(255) NOT NULL COMMENT '用户登录树洞时需要使用的密码',
  `email` varchar(50) DEFAULT NULL COMMENT '用户在注册时提供的邮箱地址',
  `phone` varchar(20) NOT NULL COMMENT '用户在注册时提供的手机号码',
  `gender` enum('F','M','Nil') NOT NULL DEFAULT 'Nil' COMMENT '用户的性别，可以使用枚举类型（0男、1女、2保密）',
  `avatar` varchar(255) DEFAULT NULL COMMENT '用户上传或选择的头像图片的文件路径或URL地址',
  `status` int(1) unsigned zerofill NOT NULL DEFAULT '0' COMMENT '用户的状态: 0: 禁用  1: 启用',
  `tree_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '用户发布的树洞数量',
  `like_count` int(11) NOT NULL DEFAULT '0' COMMENT '用户所有树洞的点赞总数',
  `is_banned` int(1) NOT NULL DEFAULT '0' COMMENT '用户是否被管理员禁言 0: 无禁言 1: 已禁言',
  `is_blocked` int(4) NOT NULL DEFAULT '0' COMMENT '用户是否被管理员封号 0: 无封号 1: 已封号',
  `created_at` bigint(20) NOT NULL COMMENT '创建时间',
  `updated_at` bigint(20) NOT NULL COMMENT '更新时间',
  `register_at` bigint(20) NOT NULL COMMENT '用户在树洞中注册的时间，使用日期时间类型来表示',
  `last_login_time` bigint(20) DEFAULT NULL COMMENT '用户最近一次登录树洞的时间，使用日期时间类型来表示',
  `is_admin` int(1) NOT NULL DEFAULT '0' COMMENT '是否为管理员用户 0: 不是 1: 是',
  `is_del` int(1) NOT NULL DEFAULT '0' COMMENT '用户是否注销 true=正常 false=注销',
  `description` varchar(255) DEFAULT '该用户未设置签名' COMMENT '用户签名或描述信息',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE KEY `username` (`username`) USING BTREE,
  UNIQUE KEY `phone` (`phone`) USING BTREE,
  UNIQUE KEY `email` (`email`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 ROW_FORMAT=DYNAMIC;
```

| 字段名          | 类型                  | 是否允许为空 | 默认值         | 主键/索引   | 注释                                                       |
| --------------- | --------------------- | ------------ | -------------- | ----------- | ---------------------------------------------------------- |
| id              | INT                   | NOT NULL     | AUTO_INCREMENT | PRIMARY KEY | 唯一标识一个用户的编号，通常使用自增长整数来表示           |
| nickname        | varchar(50)           | NOT NULL     |                | UNIQUE      | 用户在树洞中使用的昵称，可以包含中英文字符和数字           |
| password        | varchar(100)          | NOT NULL     |                |             | 用户登录时需要使用的密码，需要使用加密算法对其进行加密存储 |
| register_time   | bigint(20)            | NOT NULL     |                |             | 用户在树洞中注册的时间，使用日期时间类型来表示             |
| last_login_time | bigint(20)            |              |                |             | 用户最近一次登录树洞的时间，使用日期时间类型来表示         |
| email           | varchar(50)           |              |                | UNIQUE      | 用户在注册时提供的邮箱地址，用于找回密码等操作             |
| phone           | varchar(20)           | NOT NULL     |                | UNIQUE      | 用户在注册时提供的手机号码，用于短信验证等操作             |
| gender          | ENUM('F', 'M', 'Nil') | NOT NULL     | 2              |             | 用户的性别，可以使用枚举类型（M男、F女、Nil保密）来表示    |
| avatar          | varchar(200)          |              | default.jpg    |             | 用户上传或选择的头像图片的文件路径或URL地址                |
| tree_count      | INT                   |              | 0              |             | 用户在树洞中发布的文章数量                                 |
| like_count      | INT                   |              | 0              |             | 用户在树洞中收到的点赞数量                                 |
| is_banned       | int                   | NOT NULL     | 0              |             | 用户是否被管理员禁言，使用int来表示 0=正常 1=禁言          |
| is_blocked      | int                   | NOT NULL     | 0              |             | 用户是否被管理员封号，使用int来表示 0=正常 1=封号          |
| is_admin        | int                   | NOT NULL     | 0              |             | 用户是否为管理员,用int来表示 0=非管理员 1=管理员           |
| is_del          | int                   | NOT NULL     | 0              |             | 用户是否已经注销,用int来表示 0=正常  1=注销                |

### 树洞表结构

```sql
CREATE TABLE `treeholes` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '默认id',
  `tree_id` varchar(255) NOT NULL COMMENT '树洞唯一标识',
  `title` varchar(32) NOT NULL COMMENT '树洞标题',
  `content` text NOT NULL COMMENT '树洞内容',
  `user_id` int(11) NOT NULL COMMENT '发布该树洞的用户id',
  `status` int(2) NOT NULL DEFAULT '0' COMMENT '树洞状态 0:正常 1：删除',
  `views` bigint(20) DEFAULT '0' COMMENT '树洞浏览数',
  `likes` bigint(10) DEFAULT '0' COMMENT '树洞点赞数',
  `hot_ness` int(255) DEFAULT '0' COMMENT '树洞热度',
  `created_at` bigint(20) NOT NULL COMMENT '树洞创建时间',
  `deleted_at` bigint(20) NOT NULL COMMENT '树洞删除时间',
  `updated_at` bigint(20) NOT NULL COMMENT '树洞删除时间',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE KEY `idx_tree_id` (`tree_id`) USING BTREE,
  KEY `idx_tree_user` (`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4;
```

| 字段名        | 数据类型     | 约束                        | 描述                           |
| ------------- | ------------ | --------------------------- | ------------------------------ |
| id            | int          | PRIMARY KEY, AUTO_INCREMENT | 树洞的自增id     |
| treehole_id            | string          | NOT NULL | 树洞的唯一标识符               |
| title         | varchar(255) | NOT NULL                    | 树洞的主题或标题               |
| content       | text         | NOT NULL                    | 树洞的内容                     |
| created_at    | datetime     | NOT NULL                    | 树洞创建的时间                 |
| updated_at    | datetime     | NOT NULL                    | 树洞最后更新的时间             |
| user_id       | int          | NOT NULL                    | 发布树洞的用户ID               |
| category_id   | int          | NOT NULL                    | 树洞的分类ID                   |
| status        | tinyint(1)   | DEFAULT 1                   | 树洞的状态：1为正常，0为已删除 |
| views         | int          | DEFAULT 0                   | 树洞的浏览量                   |
| likes         | int          | DEFAULT 0                   | 树洞的点赞数                   |
| receive_gifts | int          | DEFAULT 0                   | 树洞帖子的收礼数量             |

### 礼物表结构

```sql
CREATE TABLE gifts (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,  -- 礼物的唯一 id
    name VARCHAR(255) NOT NULL,  -- 礼物的名称
    description TEXT,  -- 礼物的详细描述
    image_url VARCHAR(255),  -- 礼物的图片 URL
    price DECIMAL(8, 2) NOT NULL DEFAULT 0.0,  -- 礼物的价格，默认为 0
    created_at DATETIME NOT NULL,  -- 礼物的创建时间
    updated_at DATETIME NOT NULL  -- 礼物的最后更新时间
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

| 字段名      | 数据类型      | 是否非空 | 默认值   | 描述               |
| ----------- | ------------- | -------- | -------- | ------------------ |
| id          | INT UNSIGNED  | 是       | 自动递增 | 礼物的唯一 id      |
| name        | VARCHAR(255)  | 是       | 无       | 礼物的名称         |
| description | TEXT          | 否       | 无       | 礼物的详细描述     |
| image_url   | VARCHAR(255)  | 否       | 无       | 礼物的图片 URL     |
| price       | DECIMAL(8, 2) | 是       | 0.0      | 礼物的价格         |
| created_at  | DATETIME      | 是       | 无       | 礼物的创建时间     |
| updated_at  | DATETIME      | 是       | 无       | 礼物的最后更新时间 |