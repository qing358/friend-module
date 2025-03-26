# 游戏公会系统技术文档

## 一、系统概述

系统采用 Golang + Redis 技术栈实现。

## 二、核心功能模块

### 2.1 公会基础管理
- **功能描述**
  - 公会创建与解散
  - 公会等级与经验系统
  - 公会公告管理
  - 公会徽章管理

- **实现思路**
  ```go
  // 核心数据结构
  type Guild struct {
      ID          string    // 公会ID
      Name        string    // 公会名称
      LeaderID    string    // 会长ID
      Level       int       // 公会等级
      Exp         int64     // 公会经验
      IconURL     string    // 公会徽章
      Notice      string    // 公会公告
      CreateTime  time.Time // 创建时间
      MemberCount int       // 成员数量
      Settings    map[string]interface{} // 公会设置
  }

  // 核心实现要点
  // 1. 数据存储优化
  //    - 使用 Redis Hash 存储公会基础信息，支持部分字段更新
  //    - 使用 Redis String 存储公会名称索引，支持快速查重
  //    - 使用 Redis Sorted Set 存储公会排行榜，支持快速排序
  //    - 使用 Redis Set 存储公会标签，支持快速筛选
  // 
  // 2. 性能优化
  //    - 公会信息本地缓存，减少 Redis 访问
  //    - 使用 Redis Pipeline 批量处理独立操作（如批量更新公告）
  //    - 使用 Redis Lua 脚本处理需要原子性的复杂操作（如创建公会）
  //    - 公会数据异步持久化，减少主流程延迟
  //    - 使用 Redis Pub/Sub 实现公会信息变更通知
  // 
  // 3. 扩展性设计
  //    - 公会设置支持动态配置，无需修改数据结构
  //    - 等级系统支持自定义经验曲线
  //    - 徽章系统支持动态更新，无需重启服务
  // 
  // 4. 数据一致性
  //    - 使用 Redis 事务保证创建公会的原子性
  //    - 使用 Lua 脚本保证经验计算的原子性
  //    - 定期数据校验，确保数据一致性
  ```

### 2.2 成员管理系统
- **功能描述**
  - 成员职位体系（会长、副会长、精英、普通、见习）
  - 成员权限管理
  - 申请加入机制
  - 成员贡献度系统

- **实现思路**
  ```go
  // 核心数据结构
  type GuildMember struct {
      GuildID       string       // 公会ID
      PlayerID      string       // 玩家ID
      Position      GuildPosition // 职位
      JoinTime      time.Time    // 加入时间
      LastActive    time.Time    // 最后活跃时间
      Contribution  int64        // 贡献值
      Permissions   []string     // 自定义权限列表
      Tags          []string     // 成员标签
  }

  // 核心实现要点
  // 1. 数据存储优化
  //    - 使用 Redis Hash 存储成员信息，支持部分字段更新
  //    - 使用 Redis Sorted Set 存储成员贡献排行
  //    - 使用 Redis Set 存储成员标签，支持快速筛选
  //    - 使用 Redis List 存储申请记录，支持分页查询
  // 
  // 2. 权限系统优化
  //    - 基于 RBAC 的权限模型，支持角色继承
  //    - 权限缓存机制，减少权限检查开销
  //    - 支持自定义权限组，灵活配置权限
  //    - 权限变更实时通知，保证权限一致性
  // 
  // 3. 申请系统优化
  //    - 申请队列使用 Redis List，支持优先级
  //    - 申请状态使用 Redis Hash，支持状态追踪
  //    - 自动清理过期申请，避免数据膨胀
  //    - 申请处理支持批量操作，提高效率
  // 
  // 4. 贡献度系统优化
  //    - 使用 Redis Sorted Set 实现实时排行
  //    - 贡献度计算支持多种来源
  //    - 定期贡献度衰减，保持活跃度
  //    - 贡献度奖励支持自动发放
  ```

### 2.3 资源管理系统
- **功能描述**
  - 公会资金管理（捐献、使用、记录）
  - 公会仓库系统（物品存储、权限控制）
  - 资源使用记录
  - 资源分配机制

- **实现思路**
  ```go
  // 核心数据结构
  type GuildFund struct {
      GuildID     string    // 公会ID
      Balance     int64     // 当前余额
      TotalIncome int64     // 总收入
      TotalOut    int64     // 总支出
      DailyLimit  int64     // 每日限额
      LastReset   time.Time // 上次重置时间
  }

  type GuildWarehouseItem struct {
      GuildID    string    // 公会ID
      ItemID     string    // 物品ID
      Count      int64     // 数量
      Quality    int       // 品质
      ExpireTime time.Time // 过期时间
      Tags       []string  // 物品标签
      Metadata   map[string]interface{} // 物品元数据
  }

  // 核心实现要点
  // 1. 资金系统优化
  //    - 使用 Redis Hash 存储资金信息
  //    - 使用 Redis List 存储交易记录
  //    - 资金操作使用 Lua 脚本保证原子性
  //    - 支持资金限额和自动重置
  // 
  // 2. 仓库系统优化
  //    - 使用 Redis Hash 存储物品信息
  //    - 使用 Redis Sorted Set 实现物品排序
  //    - 使用 Redis Set 存储物品标签
  //    - 支持物品分类和快速检索
  // 
  // 3. 性能优化
  //    - 仓库数据本地缓存
  //    - 批量操作使用 Pipeline
  //    - 异步处理物品过期
  //    - 定期清理过期数据
  // 
  // 4. 扩展性设计
  //    - 支持自定义物品属性
  //    - 支持物品标签系统
  //    - 支持物品元数据扩展
  //    - 支持自定义物品分类
  ```

## 三、技术实现要点

### 3.1 数据存储设计
```go
// Redis Key 设计
const (
    KeyGuildPrefix     = "guild:"           // 公会基本信息
    KeyGuildNamePrefix = "guild:name:"      // 公会名称索引
    KeyGuildMemberPrefix = "guild:member:"  // 公会成员
    KeyPlayerGuildPrefix = "player:guild:"  // 玩家公会关系
    KeyGuildFundPrefix = "guild:fund:"      // 公会资金
    KeyGuildWarehousePrefix = "guild:warehouse:" // 公会仓库
    KeyGuildRankPrefix = "guild:rank:"      // 公会排行
    KeyGuildTagPrefix = "guild:tag:"        // 公会标签
    KeyGuildSettingPrefix = "guild:setting:" // 公会设置
)
```

### 3.2 核心实现机制
1. **原子性保证**
   - 使用 Redis Lua 脚本实现复杂操作的原子性（如创建公会、资金操作）
   - 使用 Redis 事务保证简单操作的原子性
   - 使用 Redis 分布式锁保证并发安全
   - 使用 Redis 原子计数器保证数据一致性

2. **权限控制**
   - 基于 RBAC 的权限模型
   - 权限缓存机制
   - 权限变更实时通知
   - 支持自定义权限组

3. **数据一致性**
   - 使用 Redis 事务保证数据一致性
   - 使用 Lua 脚本保证复杂操作的原子性
   - 定期数据校验和修复
   - 数据变更实时同步

4. **性能优化**
   - 合理使用 Redis 数据结构
   - 本地缓存热点数据
   - 使用 Redis Pipeline 批量处理独立操作
   - 使用 Redis Lua 脚本处理需要原子性的复杂操作
   - 异步处理非关键操作
   - 定期清理过期数据

