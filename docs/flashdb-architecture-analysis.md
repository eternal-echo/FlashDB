# FlashDB 核心架构与设计原理详解

## 概述

FlashDB是一个专为嵌入式系统设计的超轻量级数据库，具有零RAM占用、掉电保护、磨损均衡等特性。本文档详细分析其核心架构和设计原理。

## 整体架构

FlashDB采用分层架构设计，从上到下依次为：

### 1. API层
- **KVDB API**: 提供键值对数据库功能
  - `fdb_kv_set()` / `fdb_kv_get()` / `fdb_kv_del()`: 字符串类型KV操作
  - `fdb_kv_set_blob()` / `fdb_kv_get_blob()`: 二进制类型KV操作
  - `fdb_kv_iterator()`: KV遍历功能

- **TSDB API**: 提供时间序列数据库功能
  - `fdb_tsl_append()`: 添加时间序列日志
  - `fdb_tsl_iter()`: 时间序列遍历
  - `fdb_tsl_query_count()`: 查询统计

- **数据库管理**: 数据库生命周期管理
  - `fdb_kvdb_init()` / `fdb_tsdb_init()`: 初始化
  - `fdb_control()`: 参数控制

### 2. 核心模块层

#### KVDB核心模块 (fdb_kvdb.c)
- **KV查找**: 高效的键值查找算法，支持缓存和扇区遍历
- **KV创建**: 空间分配和状态流转管理
- **KV删除**: 两阶段删除（预删除→删除），确保原子性
- **垃圾回收**: 自动GC机制，回收无效数据，实现磨损均衡
- **缓存管理**: 多级缓存（KV缓存表、扇区缓存表），提升性能

#### TSDB核心模块 (fdb_tsdb.c)
- **TSL添加**: 时间戳管理和索引写入
- **TSL查询**: 基于时间范围的高效查询
- **循环覆盖**: rollover模式，自动覆盖最旧数据
- **索引管理**: 索引结构设计，支持快速定位
- **状态管理**: 支持用户自定义状态，方便数据管理

### 3. 公共底层模块

#### 扇区管理
- 扇区格式化和状态流转
- 头部信息管理（魔数、时间戳等）
- 扇区组合支持（处理大数据）

#### 状态管理
- 状态表机制：通过状态位精确控制数据生命周期
- 写粒度对齐：适配不同Flash的写入特性
- 原子性保证：确保状态流转的原子性

#### 磨损均衡
- 最旧地址（oldest_addr）管理
- 循环写入策略
- 扇区轮转机制

#### 掉电保护
- 状态原子性：通过状态表确保操作原子性
- 启动恢复检查：检测异常状态并自动恢复
- 数据一致性保证

#### Flash操作
- 读写擦除基础操作
- 写粒度处理（1/8/32/64/128位）
- CRC32校验确保数据完整性

## 核心数据结构

### KVDB结构 (fdb_kvdb)
```c
struct fdb_kvdb {
    struct fdb_db parent;              // 继承基础数据库结构
    struct fdb_default_kv default_kvs; // 默认KV配置
    bool gc_request;                   // GC请求标志
    struct fdb_kv_cache_node kv_cache_table[FDB_KV_CACHE_TABLE_SIZE];
    struct kvdb_sec_info sector_cache_table[FDB_SECTOR_CACHE_TABLE_SIZE];
};
```

### TSDB结构 (fdb_tsdb)
```c
struct fdb_tsdb {
    struct fdb_db parent;              // 继承基础数据库结构
    struct tsdb_sec_info cur_sec;      // 当前扇区信息
    fdb_time_t last_time;              // 最后时间戳
    fdb_get_time get_time;             // 时间获取函数
    uint32_t max_len;                  // 最大日志长度
    bool rollover;                     // 是否启用循环模式
};
```

### KV节点结构
```c
struct kv_hdr_data {
    uint8_t status_table[KV_STATUS_TABLE_SIZE]; // 状态表
    uint32_t magic;                             // 魔数 (KV40)
    uint32_t len;                               // 节点总长度
    uint32_t crc32;                             // CRC32校验
    uint8_t name_len;                           // 键名长度
    uint32_t value_len;                         // 值长度
};
```

### TSL节点结构
```c
struct log_idx_data {
    uint8_t status_table[TSL_STATUS_TABLE_SIZE]; // 状态表
    fdb_time_t time;                             // 时间戳
    uint32_t log_len;                            // 日志长度
    uint32_t log_addr;                           // 日志地址
};
```

## 状态机与生命周期

### KV状态流转
1. **UNUSED** → **PRE_WRITE**: 开始写入
2. **PRE_WRITE** → **WRITE**: 写入完成
3. **WRITE** → **PRE_DELETE**: 开始删除
4. **PRE_DELETE** → **DELETED**: 删除完成

### 扇区状态流转
1. **UNUSED** → **EMPTY**: 格式化完成
2. **EMPTY** → **USING**: 开始使用
3. **USING** → **FULL**: 空间耗尽
4. **FULL** → **GC**: 标记为GC
5. **GC** → **EMPTY**: GC完成，重新格式化

## 核心机制详解

### 1. 垃圾回收机制 (GC)

#### 触发条件
- 空间不足时自动触发
- 可用扇区数量低于阈值
- 手动触发GC请求

#### GC流程
1. **标记阶段**: 标记需要回收的扇区（状态为DIRTY_TRUE）
2. **迁移阶段**: 将有效数据迁移到新扇区
3. **回收阶段**: 格式化旧扇区，释放空间
4. **更新阶段**: 更新oldest_addr指针

#### 关键代码流程
```c
static bool do_gc(kv_sec_info_t sector, void *arg1, void *arg2) {
    // 1. 标记扇区为GC状态
    _fdb_write_status(db, sector->addr + SECTOR_DIRTY_OFFSET, 
                      status_table, FDB_SECTOR_DIRTY_STATUS_NUM, 
                      FDB_SECTOR_DIRTY_GC, true);
    
    // 2. 遍历并迁移有效KV
    kv.addr.start = sector->addr + SECTOR_HDR_DATA_SIZE;
    do {
        read_kv(db, &kv);
        if (kv.crc_is_ok && (kv.status == FDB_KV_WRITE || kv.status == FDB_KV_PRE_DELETE)) {
            move_kv(db, &kv);  // 迁移有效KV
        }
    } while ((kv.addr.start = get_next_kv_addr(db, sector, &kv)) != FAILED_ADDR);
    
    // 3. 格式化扇区
    format_sector(db, sector->addr, SECTOR_NOT_COMBINED);
}
```

### 2. 多级缓存机制

#### KV缓存表
- 缓存最近访问的KV地址
- 使用CRC16作为快速比较
- LRU替换策略

#### 扇区缓存表
- 缓存扇区状态信息
- 减少Flash读取次数
- 提升扇区遍历性能

#### 缓存更新策略
```c
static void update_kv_cache(fdb_kvdb_t db, const char *name, size_t name_len, uint32_t addr) {
    uint16_t name_crc = (uint16_t)(fdb_calc_crc32(0, name, name_len) >> 16);
    
    // 查找已存在的缓存项或空槽位
    for (i = 0; i < FDB_KV_CACHE_TABLE_SIZE; i++) {
        if (db->kv_cache_table[i].name_crc == name_crc) {
            // 更新已存在的缓存
            db->kv_cache_table[i].addr = addr;
            return;
        }
        // 记录空槽位和最少活跃的槽位
        if (db->kv_cache_table[i].addr == FDB_DATA_UNUSED) {
            empty_index = i;
        } else if (db->kv_cache_table[i].active < min_activity) {
            min_activity_index = i;
            min_activity = db->kv_cache_table[i].active;
        }
    }
    
    // 使用空槽位或替换最少活跃的槽位
    if (empty_index < FDB_KV_CACHE_TABLE_SIZE) {
        target_index = empty_index;
    } else {
        target_index = min_activity_index;
    }
    
    db->kv_cache_table[target_index].name_crc = name_crc;
    db->kv_cache_table[target_index].addr = addr;
    db->kv_cache_table[target_index].active = 0;
}
```

### 3. 掉电保护机制

#### 状态原子性
通过状态表机制确保操作的原子性：
- 每个操作都有对应的状态位
- 状态流转确保中间状态可恢复
- Flash写入失败时状态不会改变

#### 恢复检查流程
```c
static fdb_err_t _fdb_kv_load(fdb_kvdb_t db) {
    // 1. 检查所有扇区头部信息
    sector_iterator(db, &sector, FDB_SECTOR_STORE_UNUSED, 
                   &check_failed_count, db, check_and_recovery_sector_cb, false);
    
    // 2. 恢复异常状态的KV
    sector_iterator(db, &sector, FDB_SECTOR_STORE_UNUSED, 
                   db, NULL, check_and_recovery_kv_cb, true);
    
    // 3. 恢复GC中断
    sector_iterator(db, &sector, FDB_SECTOR_STORE_UNUSED, 
                   db, NULL, check_and_recovery_gc_cb, false);
}
```

#### 异常恢复处理
```c
static bool check_and_recovery_kv_cb(fdb_kv_t kv, void *arg1, void *arg2) {
    fdb_kvdb_t db = arg1;
    
    // 恢复预删除状态的KV
    if (kv->crc_is_ok && kv->status == FDB_KV_PRE_DELETE) {
        FDB_INFO("Found an KV (%.*s) which has changed value failed. Now will recovery it.\n", 
                 kv->name_len, kv->name);
        move_kv(db, kv);  // 恢复KV
    } 
    // 标记异常状态的KV
    else if (kv->status == FDB_KV_PRE_WRITE) {
        _fdb_write_status(db, kv->addr.start, status_table, 
                         FDB_KV_STATUS_NUM, FDB_KV_ERR_HDR, true);
    }
    
    return false;
}
```

### 4. 磨损均衡机制

#### 最旧地址管理
通过`oldest_addr`指针实现循环写入：
- 始终从最旧的扇区开始分配
- GC完成后更新oldest_addr
- 确保Flash使用均匀

#### 扇区轮转策略
```c
static uint32_t get_next_sector_addr(fdb_kvdb_t db, kv_sec_info_t pre_sec, uint32_t traversed_len) {
    uint32_t cur_block_size;
    
    if (pre_sec->combined == SECTOR_NOT_COMBINED) {
        cur_block_size = db_sec_size(db);
    } else {
        cur_block_size = pre_sec->combined * db_sec_size(db);
    }
    
    if (traversed_len + cur_block_size <= db_max_size(db)) {
        // 计算下一个扇区地址
        if (pre_sec->addr + cur_block_size < db_max_size(db)) {
            return pre_sec->addr + cur_block_size;
        } else {
            // 到达末尾，回到开头
            return 0;
        }
    } else {
        return FAILED_ADDR;  // 遍历完成
    }
}
```

### 5. TSDB循环模式

#### Rollover机制
当数据库空间满时，TSDB可以自动覆盖最旧的数据：
- `rollover = true`: 启用循环模式
- 自动计算oldest_addr
- 覆盖最旧扇区的数据

#### 时间序列查询优化
```c
fdb_err_t fdb_tsl_iter_by_time(fdb_tsdb_t db, fdb_time_t from, fdb_time_t to,
                               fdb_tsl_cb cb, void *cb_arg) {
    // 1. 定位起始扇区
    // 2. 在扇区内二分查找起始位置
    // 3. 按时间顺序遍历
    // 4. 过滤时间范围和状态
}
```

## 存储布局

### Flash分区布局
```
+----------+----------+----------+----------+
| Sector 0 | Sector 1 | Sector 2 | Sector N |
| (Empty)  | (Using)  | (Full)   | (GC)     |
+----------+----------+----------+----------+
     ↑
oldest_addr (轮转指针)
```

### 扇区内部结构
```
+----------------+-------+-------+-------+-------+
| Sector Header  |  KV1  |  KV2  |  KV3  |  ...  |
| (Magic,Status) |       |       |       |       |
+----------------+-------+-------+-------+-------+
```

### KV节点布局
```
+--------+-------+--------+--------+------+-------+
| Status | Magic | Length | CRC32  | Name | Value |
+--------+-------+--------+--------+------+-------+
```

## 关键路径分析

### 读路径: API → 缓存查找 → 扇区遍历 → Flash读取
1. 检查KV缓存表
2. 缓存命中直接返回
3. 缓存未命中则遍历扇区
4. 更新缓存并返回

### 写路径: API → 空间分配 → 状态流转 → Flash写入 → 缓存更新
1. 分配KV空间
2. 写入PRE_WRITE状态
3. 写入数据内容
4. 写入WRITE状态
5. 更新缓存

### GC路径: 触发条件 → 标记扇区 → 迁移有效数据 → 格式化扇区
1. 检查触发条件
2. 标记脏扇区
3. 迁移有效数据
4. 格式化并更新oldest_addr

### 恢复路径: 启动检查 → 状态修复 → 数据恢复 → 缓存重建
1. 检查所有扇区
2. 修复异常状态
3. 恢复中断操作
4. 重建缓存表

## 配置与优化

### 关键配置参数
- `FDB_WRITE_GRAN`: Flash写粒度（1/8/32/64/128位）
- `FDB_KV_CACHE_TABLE_SIZE`: KV缓存表大小
- `FDB_SECTOR_CACHE_TABLE_SIZE`: 扇区缓存表大小
- `FDB_GC_EMPTY_SEC_THRESHOLD`: GC触发阈值

### 性能优化建议
1. 合理设置缓存大小
2. 根据应用选择合适的扇区大小
3. 启用缓存功能提升查找性能
4. 定期执行GC避免空间不足

## 总结

FlashDB通过精心设计的分层架构、状态机制、缓存策略和保护机制，实现了一个高可靠、高性能的嵌入式数据库。其核心特性包括：

1. **零RAM占用**: 所有数据直接存储在Flash中
2. **掉电保护**: 通过状态机制确保数据一致性
3. **磨损均衡**: 循环写入延长Flash寿命
4. **高性能**: 多级缓存机制提升访问速度
5. **易移植**: 简洁的接口设计，便于跨平台移植

这使得FlashDB特别适合资源受限的嵌入式系统，在保证可靠性的同时提供了优秀的性能表现。
