# Redis 缓存 Key 删除指南

基于你的项目中Redis的使用情况，以下是删除特定前缀缓存key的几种方法：

## 方法一：使用 Redis CLI 命令

### 1. 删除特定前缀的所有 keys（推荐）

```bash
# 删除以 "your_prefix:" 开头的所有 keys
redis-cli --scan --pattern "your_prefix:*" | xargs redis-cli del

# 或者使用 bash 管道（更安全，支持大量key）
redis-cli --scan --pattern "your_prefix:*" | while read key; do
    redis-cli del "$key"
done
```

### 2. 基于你项目的具体前缀

根据你的代码，keys 格式为 `{memory_index}:{vec_num}` 和 `{memory_index}-vec_num`

```bash
# 删除特定 memory_index 的所有相关 keys
redis-cli --scan --pattern "your_memory_index:*" | xargs redis-cli del
redis-cli --scan --pattern "your_memory_index-*" | xargs redis-cli del

# 或者一次性删除（如果确定前缀）
redis-cli --scan --pattern "your_memory_index*" | xargs redis-cli del
```

## 方法二：使用 Python 脚本

### 1. 基于现有的 RedisMemory 类

```python
from autogpt.memory.redismem import RedisMemory
from autogpt.config import Config

# 初始化配置和Redis连接
cfg = Config()
redis_memory = RedisMemory(cfg)

# 删除特定前缀的keys
def delete_keys_with_prefix(redis_client, prefix):
    """删除具有特定前缀的所有keys"""
    keys_to_delete = []
    cursor = 0
    
    while True:
        cursor, keys = redis_client.scan(cursor=cursor, match=f"{prefix}*", count=1000)
        if keys:
            keys_to_delete.extend(keys)
        if cursor == 0:
            break
    
    if keys_to_delete:
        deleted_count = redis_client.delete(*keys_to_delete)
        print(f"删除了 {deleted_count} 个keys")
        return deleted_count
    else:
        print("没有找到匹配的keys")
        return 0

# 使用示例
delete_keys_with_prefix(redis_memory.redis, "your_prefix:")
```

### 2. 更安全的批量删除脚本

```python
import redis

def safe_delete_keys_with_prefix(redis_client, prefix, batch_size=1000):
    """安全地删除具有特定前缀的keys，分批处理"""
    total_deleted = 0
    cursor = 0
    
    print(f"开始删除前缀为 '{prefix}' 的keys...")
    
    while True:
        cursor, keys = redis_client.scan(
            cursor=cursor, 
            match=f"{prefix}*", 
            count=batch_size
        )
        
        if keys:
            # 分批删除以避免阻塞
            batch_deleted = redis_client.delete(*keys)
            total_deleted += batch_deleted
            print(f"本批删除了 {batch_deleted} 个keys，总计删除 {total_deleted} 个")
        
        if cursor == 0:
            break
    
    print(f"删除完成，总共删除了 {total_deleted} 个keys")
    return total_deleted

# 使用示例
r = redis.Redis(host='localhost', port=6379, db=0)
safe_delete_keys_with_prefix(r, "your_prefix:")
```

## 方法三：使用 Redis 原生命令

### 1. 直接在 Redis CLI 中执行

```redis
# 进入 Redis CLI
redis-cli

# 查看匹配的keys（先预览）
SCAN 0 MATCH your_prefix:* COUNT 10

# 使用 EVAL 脚本删除（推荐用于大量keys）
EVAL "
local keys = redis.call('SCAN', 0, 'MATCH', ARGV[1], 'COUNT', 1000)
local deleted = 0
for i=1, #keys[2] do
    redis.call('DEL', keys[2][i])
    deleted = deleted + 1
end
return deleted
" 0 your_prefix:*
```

### 2. 使用 Redis 的 UNLINK 命令（非阻塞删除）

```bash
# 对于大量数据，使用 UNLINK 而不是 DEL，避免阻塞
redis-cli --scan --pattern "your_prefix:*" | xargs redis-cli unlink
```

## 方法四：针对你项目的专用方法

基于你的 `RedisMemory` 类，你可以扩展一个专门的清理方法：

```python
def clear_memory_index(self, memory_index: str) -> str:
    """
    清除特定 memory_index 的所有数据
    
    Args:
        memory_index: 要清除的内存索引名称
    
    Returns: 清除结果消息
    """
    # 删除所有相关的hash keys
    keys_pattern = f"{memory_index}:*"
    cursor = 0
    deleted_count = 0
    
    while True:
        cursor, keys = self.redis.scan(cursor=cursor, match=keys_pattern, count=1000)
        if keys:
            deleted_count += self.redis.delete(*keys)
        if cursor == 0:
            break
    
    # 删除计数器
    self.redis.delete(f"{memory_index}-vec_num")
    
    # 尝试删除索引
    try:
        self.redis.ft(memory_index).drop_index(delete_documents=True)
    except Exception as e:
        print(f"索引删除失败（可能不存在）: {e}")
    
    return f"已删除 memory_index '{memory_index}' 的 {deleted_count} 个keys"
```

## 注意事项

1. **生产环境警告**：在生产环境中删除缓存数据前，请务必备份重要数据
2. **性能考虑**：对于大量keys，使用SCAN而不是KEYS命令，避免阻塞Redis服务
3. **原子性**：如果需要原子操作，考虑使用Redis事务或Lua脚本
4. **验证**：删除前可以先用SCAN预览要删除的keys

## 常用命令速查

```bash
# 查看所有keys（小心使用）
redis-cli keys "*"

# 查看特定前缀的keys数量
redis-cli eval "return #redis.call('SCAN', 0, 'MATCH', ARGV[1])[2]" 0 "your_prefix:*"

# 批量删除（安全版本）
redis-cli --scan --pattern "your_prefix:*" | head -100 | xargs redis-cli del
```

选择最适合你当前需求的方法即可！