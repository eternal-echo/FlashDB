# FlashDB 核心架构快速指南

> 🎯 这是对FlashDB核心架构的精简总结，详细分析请查看 [技术分析文档索引](./docs/technical-analysis-index.md)

## 架构概览

```
┌─────────────────────────────────────────────────────────┐
│                     API层 (flashdb.h)                   │
├─────────────────────┬───────────────────────────────────┤
│    KVDB API         │        TSDB API                   │
│  · fdb_kv_set()     │    · fdb_tsl_append()            │
│  · fdb_kv_get()     │    · fdb_tsl_iter()              │
│  · fdb_kv_del()     │    · fdb_tsl_query()             │
├─────────────────────┴───────────────────────────────────┤
│              核心模块层 (fdb_kvdb.c/fdb_tsdb.c)         │
├─────────────────────┬───────────────────────────────────┤
│   KVDB核心模块       │       TSDB核心模块                │
│ · KV查找/创建/删除   │   · TSL添加/查询                  │
│ · 垃圾回收(GC)      │   · 循环覆盖                      │
│ · 缓存管理          │   · 索引管理                      │
├─────────────────────┴───────────────────────────────────┤
│                    公共底层模块                          │
│ · 扇区管理  · 状态管理  · 磨损均衡  · 掉电保护  · Flash操作 │
└─────────────────────────────────────────────────────────┘
```

## 核心特性

### 🔋 零RAM占用
- 直接操作Flash存储，无需RAM缓存
- 状态表机制管理数据生命周期
- 适合资源受限的嵌入式环境

### ⚡ 掉电保护
- 状态原子性：操作要么完全成功，要么完全回滚
- 启动恢复检查：自动检测并修复异常状态
- CRC32校验：确保数据完整性

### 🔄 磨损均衡
- 最旧地址管理：自动轮转写入位置
- 循环写入策略：均匀分布Flash磨损
- 扇区轮转机制：延长Flash使用寿命

### 🗑️ 自动垃圾回收
- 后台GC：自动回收无效数据空间
- 智能触发：基于空闲空间阈值
- 原子操作：GC过程确保数据安全

## 数据库类型

### KVDB (键值数据库)
```c
// 字符串类型
fdb_kv_set("name", "FlashDB");
char *value = fdb_kv_get("name");

// 二进制类型  
struct config cfg = {0};
fdb_kv_set_blob("config", &cfg, sizeof(cfg));
```

**应用场景**: 配置存储、参数管理、状态保存

### TSDB (时间序列数据库)
```c
// 添加日志
struct log_data data = {temp, humidity, timestamp};
fdb_tsl_append(&data, sizeof(data));

// 查询数据
fdb_tsl_iter_by_time(start_time, end_time, callback);
```

**应用场景**: 传感器日志、历史数据、事件记录

## 关键机制

### 状态表机制
```
状态流转: 空闲 → 写入中 → 写入完成 → 预删除 → 删除
每个状态对应特定的状态位组合，确保操作原子性
```

### 扇区管理
```
扇区状态: 空 → 空闲 → 使用中 → 满 → GC
自动格式化和头部信息管理，支持扇区组合处理大数据
```

### 缓存机制
```
多级缓存:
├── KV缓存表: 缓存最近访问的KV项
└── 扇区缓存表: 缓存扇区统计信息
```

## 性能特点

| 指标 | KVDB | TSDB |
|------|------|------|
| 写入性能 | 快速索引 | 顺序追加 |
| 查询性能 | 哈希+缓存 | 时间索引 |
| 空间效率 | 按需分配 | 循环覆盖 |
| GC开销 | 自动触发 | 循环模式低 |

## 移植适配

FlashDB设计了标准化的移植接口：

```c
// 必需实现的移植层函数
fdb_err_t fdb_flash_read(fdb_db_t db, uint32_t addr, void *buf, size_t size);
fdb_err_t fdb_flash_write(fdb_db_t db, uint32_t addr, const void *buf, size_t size);
fdb_err_t fdb_flash_erase(fdb_db_t db, uint32_t addr, size_t size);
```

支持多种Flash类型：NOR Flash、NAND Flash、eMMC等

## 快速上手

1. **查看详细分析**: [FlashDB架构分析文档](./docs/flashdb-architecture-analysis.md)
2. **查看架构图**: [FlashDB核心架构图](./docs/flashdb.drawio)
3. **性能分析**: [基准测试分析](./docs/bench.drawio)
4. **完整索引**: [技术分析文档索引](./docs/technical-analysis-index.md)

---
*这是对FlashDB核心架构的快速概览，更多详细内容请查看完整的技术分析文档。*
