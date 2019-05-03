# 1. Strings

| 命令 | 格式 | 描述 | 
| :-- | :-- | :-- |
| [APPEND](https://redis.io/commands/append) | `APPEND key value` | Append a value to a key | 
| [BITCOUNT](https://redis.io/commands/bitcount) | `BITCOUNT key [start end]` | Count set bits in a string | 
| [BITFIELD](https://redis.io/commands/bitfield) | `BITFIELD key `<br>`[GET type offset] `<br>`[SET type offset value] `<br>`[INCRBY type offset increment] `<br>`[OVERFLOW WRAP\|SAT\|FAIL]` | Perform arbitrary bitfield integer operations on strings | 
| [BITOP](https://redis.io/commands/bitop) | `BITOP operation destkey key [key ...]` | Perform bitwise operations between strings | 
| [BITPOS](https://redis.io/commands/bitpos) | `BITPOS key bit [start] [end]` | Find first bit set or clear in a string | 
| [DECR](https://redis.io/commands/decr) | `DECR key` | Decrement the integer value of a key by one | 
| [DECRBY](https://redis.io/commands/decrby) | `DECRBY key decrement` | Decrement the integer value of a key by the given number | 
| [GET](https://redis.io/commands/get) | `GET key` | Get the value of a key | 
| [GETBIT](https://redis.io/commands/getbit) | `GETBIT key offset` | Returns the bit value at offset in the string value stored at key | 
| [GETRANGE](https://redis.io/commands/getrange) | `GETRANGE key start end` | Get a substring of the string stored at a key | 
| [GETSET](https://redis.io/commands/getset) | `GETSET key value` | Set the string value of a key and return its old value | 
| [INCR](https://redis.io/commands/incr) | `INCR key` | Increment the integer value of a key by one | 
| [INCRBY](https://redis.io/commands/incrby) | `INCRBY key increment` | Increment the integer value of a key by the given amount | 
| [INCRBYFLOAT](https://redis.io/commands/incrbyfloat) | `INCRBYFLOAT key increment` | Increment the float value of a key by the given amount | 
| [MGET](https://redis.io/commands/mget) | `MGET key [key ...]` | Get the values of all the given keys | 
| [MSET](https://redis.io/commands/mset) | `MSET key value [key value ...]` | Set multiple keys to multiple values | 
| [MSETNX](https://redis.io/commands/msetnx) | `MSETNX key value [key value ...]` | Set multiple keys to multiple values, <br>only if none of the keys exist | 
| [PSETEX](https://redis.io/commands/psetex) | `PSETEX key milliseconds value` | Set the value and expiration in milliseconds of a key | 
| [SET](https://redis.io/commands/set) | `SET key value `<br>`[expiration EX seconds\|PX milliseconds] `<br>`[NX\|XX]` | Set the string value of a key | 
| [SETBIT](https://redis.io/commands/setbit) | `SETBIT key offset value` | Sets or clears the bit at offset in the string value stored at key | 
| [SETEX](https://redis.io/commands/setex) | `SETEX key seconds value` | Set the value and expiration of a key | 
| [SETNX](https://redis.io/commands/setnx) | `SETNX key value` | Set the value of a key, <br>only if the key does not exist | 
| [SETRANGE](https://redis.io/commands/setrange) | `SETRANGE key offset value` | Overwrite part of a string at key starting at the specified offset | 
| [STRLEN](https://redis.io/commands/strlen) | `STRLEN key` | Get the length of the value stored in a key | 

# 2. Lists

| 命令 | 格式 | 描述 | 
| :-- | :-- | :-- |
| [BLPOP](https://redis.io/commands/blpop) | `BLPOP key [key ...] timeout` | Remove and get the first element in a list, <br>or block until one is available |
| [BRPOP](https://redis.io/commands/brpop) | `BRPOP key [key ...] timeout` | Remove and get the last element in a list, <br>or block until one is available |
| [BRPOPLPUSH](https://redis.io/commands/brpoplpush) | `BRPOPLPUSH source destination timeout` | Pop a value from a list, <br>push it to another list and return it; <br>or block until one is available |
| [LINDEX](https://redis.io/commands/lindex) | `LINDEX key index` | Get an element from a list by its index |
| [LINSERT](https://redis.io/commands/linsert) | `LINSERT key BEFORE\|AFTER pivot value` | Insert an element before or after another element in a list |
| [LLEN](https://redis.io/commands/llen) | `LLEN key` | Get the length of a list |
| [LPOP](https://redis.io/commands/lpop) | `LPOP key` | Remove and get the first element in a list |
| [LPUSH](https://redis.io/commands/lpush) | `LPUSH key value [value ...]` | Prepend one or multiple values to a list |
| [LPUSHX](https://redis.io/commands/lpushx) | `LPUSHX key value` | Prepend a value to a list, <br>only if the list exists |
| [LRANGE](https://redis.io/commands/lrange) | `LRANGE key start stop` | Get a range of elements from a list |
| [LREM](https://redis.io/commands/lrem) | `LREM key count value` | Remove elements from a list |
| [LSET](https://redis.io/commands/lset) | `LSET key index value` | Set the value of an element in a list by its index |
| [LTRIM](https://redis.io/commands/ltrim) | `LTRIM key start stop` | Trim a list to the specified range |
| [RPOP](https://redis.io/commands/rpop) | `RPOP key` | Remove and get the last element in a list |
| [RPOPLPUSH](https://redis.io/commands/rpoplpush) | `RPOPLPUSH source destination` | Remove the last element in a list, <br>prepend it to another list and return it |
| [RPUSH](https://redis.io/commands/rpush) | `RPUSH key value [value ...]` | Append one or multiple values to a list |
| [RPUSHX](https://redis.io/commands/rpushx) | `RPUSHX key value` | Append a value to a list, <br>only if the list exists |

# 3. Hashes

| 命令 | 格式 | 描述 | 
| :-- | :-- | :-- |
| [HDEL](https://redis.io/commands/hdel) | `HDEL key field [field ...]` | Delete one or more hash fields |
| [HEXISTS](https://redis.io/commands/hexists) | `HEXISTS key field` | Determine if a hash field exists |
| [HGET](https://redis.io/commands/hget) | `HGET key field` | Get the value of a hash field |
| [HGETALL](https://redis.io/commands/hgetall) | `HGETALL key` | Get all the fields and values in a hash |
| [HINCRBY](https://redis.io/commands/hincrby) | `HINCRBY key field increment` | Increment the integer value of a hash field by the given number |
| [HINCRBYFLOAT](https://redis.io/commands/hincrbyfloat) | `HINCRBYFLOAT key field increment` | Increment the float value of a hash field by the given amount |
| [HKEYS](https://redis.io/commands/hkeys) | `HKEYS key` | Get all the fields in a hash |
| [HLEN](https://redis.io/commands/hlen) | `HLEN key` | Get the number of fields in a hash |
| [HMGET](https://redis.io/commands/hmget) | `HMGET key field [field ...]` | Get the values of all the given hash fields |
| [HMSET](https://redis.io/commands/hmset) | `HMSET key field value [field value ...]` | Set multiple hash fields to multiple values |
| [HSET](https://redis.io/commands/hset) | `HSET key field value` | Set the string value of a hash field |
| [HSETNX](https://redis.io/commands/hsetnx) | `HSETNX key field value` | Set the value of a hash field, <br>only if the field does not exist |
| [HSTRLEN](https://redis.io/commands/hstrlen) | `HSTRLEN key field` | Get the length of the value of a hash field |
| [HVALS](https://redis.io/commands/hvals) | `HVALS key` | Get all the values in a hash |
| [HSCAN](https://redis.io/commands/hscan) | `HSCAN key cursor [MATCH pattern] [COUNT count]` | Incrementally iterate hash fields and associated values |

# 4. Sets

| 命令 | 格式 | 描述 | 
| :-- | :-- | :-- |
| [SADD](https://redis.io/commands/sadd) | `SADD key member [member ...]` | Add one or more members to a set |
| [SCARD](https://redis.io/commands/scard) | `SCARD key` | Get the number of members in a set |
| [SDIFF](https://redis.io/commands/sdiff) | `SDIFF key [key ...]` | Subtract multiple sets |
| [SDIFFSTORE](https://redis.io/commands/sdiffstore) | `SDIFFSTORE destination key [key ...]` | Subtract multiple sets and store the resulting set in a key |
| [SINTER](https://redis.io/commands/sinter) | `SINTER key [key ...]` | Intersect multiple sets |
| [SINTERSTORE](https://redis.io/commands/sinterstore) | `SINTERSTORE destination key [key ...]` | Intersect multiple sets and store the resulting set in a key |
| [SISMEMBER](https://redis.io/commands/sismember) | `SISMEMBER key member` | Determine if a given value is a member of a set |
| [SMEMBERS](https://redis.io/commands/smembers) | `SMEMBERS key` | Get all the members in a set |
| [SMOVE](https://redis.io/commands/smove) | `SMOVE source destination member` | Move a member from one set to another |
| [SPOP](https://redis.io/commands/spop) | `SPOP key [count]` | Remove and return one or multiple random members from a set |
| [SRANDMEMBER](https://redis.io/commands/srandmember) | `SRANDMEMBER key [count]` | Get one or multiple random members from a set |
| [SREM](https://redis.io/commands/srem) | `SREM key member [member ...]` | Remove one or more members from a set |
| [SUNION](https://redis.io/commands/sunion) | `SUNION key [key ...]` | Add multiple sets |
| [SUNIONSTORE](https://redis.io/commands/sunionstore) | `SUNIONSTORE destination key [key ...]` | Add multiple sets and store the resulting set in a key |
| [SSCAN](https://redis.io/commands/sscan) | `SSCAN key cursor [MATCH pattern] [COUNT count]` | Incrementally iterate Set elements |

# 5. Sorted Sets

| 命令 | 格式 | 描述 | 
| :-- | :-- | :-- |
| [BZPOPMIN](https://redis.io/commands/bzpopmin) | `BZPOPMIN key [key ...] timeout` | Remove and return the member with the lowest score from one or more sorted sets, or block until one is available |
| [BZPOPMAX](https://redis.io/commands/bzpopmax) | `BZPOPMAX key [key ...] timeout` | Remove and return the member with the highest score from one or more sorted sets, or block until one is available |
| [ZADD](https://redis.io/commands/zadd) | `ZADD key [NX\|XX] [CH] [INCR] score member [score member ...]` | Add one or more members to a sorted set, or update its score if it already exists |
| [ZCARD](https://redis.io/commands/zcard) | `ZCARD key` | Get the number of members in a sorted set |
| [ZCOUNT](https://redis.io/commands/zcount) | `ZCOUNT key min max` | Count the members in a sorted set with scores within the given values |
| [ZINCRBY](https://redis.io/commands/zincrby) | `ZINCRBY key increment member` | Increment the score of a member in a sorted set |
| [ZINTERSTORE](https://redis.io/commands/zinterstore) | `ZINTERSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM\|MIN\|MAX]` | Intersect multiple sorted sets and store the resulting sorted set in a new key |
| [ZLEXCOUNT](https://redis.io/commands/zlexcount) | `ZLEXCOUNT key min max` | Count the number of members in a sorted set between a given lexicographical range |
| [ZPOPMAX](https://redis.io/commands/zpopmax) | `ZPOPMAX key [count]` | Remove and return members with the highest scores in a sorted set |
| [ZPOPMIN](https://redis.io/commands/zpopmin) | `ZPOPMIN key [count]` | Remove and return members with the lowest scores in a sorted set |
| [ZRANGE](https://redis.io/commands/zrange) | `ZRANGE key start stop [WITHSCORES]` | Return a range of members in a sorted set, by index |
| [ZRANGEBYLEX](https://redis.io/commands/zrangebylex) | `ZRANGEBYLEX key min max `<br>`[LIMIT offset count]` | Return a range of members in a sorted set, by lexicographical range |
| [ZREVRANGEBYLEX](https://redis.io/commands/zrevrangebylex) | `ZREVRANGEBYLEX key max min `<br>`[LIMIT offset count]` | Return a range of members in a sorted set, by lexicographical range, ordered from higher to lower strings |
| [ZRANGEBYSCORE](https://redis.io/commands/zrangebyscore) | `ZRANGEBYSCORE key min max `<br>`[WITHSCORES] [LIMIT offset count]` | Return a range of members in a sorted set, by score |
| [ZRANK](https://redis.io/commands/zrank) | `ZRANK key member` | Determine the index of a member in a sorted set |
| [ZREM](https://redis.io/commands/zrem) | `ZREM key member [member ...]` | Remove one or more members from a sorted set |
| [ZREMRANGEBYLEX](https://redis.io/commands/zremrangebylex) | `ZREMRANGEBYLEX key min max` | Remove all members in a sorted set between the given lexicographical range |
| [ZREMRANGEBYRANK](https://redis.io/commands/zremrangebyrank) | `ZREMRANGEBYRANK key start stop` | Remove all members in a sorted set within the given indexes |
| [ZREMRANGEBYSCORE](https://redis.io/commands/zremrangebyscore) | `ZREMRANGEBYSCORE key min max` | Remove all members in a sorted set within the given scores |
| [ZREVRANGE](https://redis.io/commands/zrevrange) | `ZREVRANGE key start stop [WITHSCORES]` | Return a range of members in a sorted set, by index, with scores ordered from high to low |
| [ZREVRANGEBYSCORE](https://redis.io/commands/zrevrangebyscore) | `ZREVRANGEBYSCORE key max min `<br>`[WITHSCORES] [LIMIT offset count]` | Return a range of members in a sorted set, by score, with scores ordered from high to low |
| [ZREVRANK](https://redis.io/commands/zrevrank) | `ZREVRANK key member` | Determine the index of a member in a sorted set, with scores ordered from high to low |
| [ZSCORE](https://redis.io/commands/zscore) | `ZSCORE key member` | Get the score associated with the given member in a sorted set |
| [ZUNIONSTORE](https://redis.io/commands/zunionstore) | `ZUNIONSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM\|MIN\|MAX]` | Add multiple sorted sets and store the resulting sorted set in a new key |
| [ZSCAN](https://redis.io/commands/zscan) | `ZSCAN key cursor [MATCH pattern] `<br>`[COUNT count]` |Incrementally iterate sorted sets elements and associated scores |

# 6. Keys

| 命令 | 格式 | 描述 | 
| :-- | :-- | :-- |
| [DEL](https://redis.io/commands/del) | `DEL key [key ...]` | Delete a key |
| [DUMP](https://redis.io/commands/dump) | `DUMP key` | Return a serialized version of the value stored at the specified key |
| [EXISTS](https://redis.io/commands/exists) | `EXISTS key [key ...]` | Determine if a key exists |
| [EXPIRE](https://redis.io/commands/expire) | `EXPIRE key seconds` | Set a key's time to live in seconds |
| [EXPIREAT](https://redis.io/commands/expireat) | `EXPIREAT key timestamp` | Set the expiration for a key as a UNIX timestamp |
| [KEYS](https://redis.io/commands/keys) | `KEYS pattern` | Find all keys matching the given pattern |
| [MIGRATE](https://redis.io/commands/migrate) | `MIGRATE host port key\|"" destination-db timeout [COPY] [REPLACE] [KEYS key [key ...]]` | Atomically transfer a key from a Redis instance to another one |
| [MOVE](https://redis.io/commands/move) | `MOVE key db` | Move a key to another database |
| [OBJECT](https://redis.io/commands/object) | `OBJECT subcommand [arguments [arguments ...]]` | Inspect the internals of Redis objects |
| [PERSIST](https://redis.io/commands/persist) | `PERSIST key` | Remove the expiration from a key |
| [PEXPIRE](https://redis.io/commands/pexpire) | `PEXPIRE key milliseconds` | Set a key's time to live in milliseconds |
| [PEXPIREAT](https://redis.io/commands/pexpireat) | `PEXPIREAT key milliseconds-timestamp` | Set the expiration for a key as a UNIX timestamp specified in milliseconds |
| [PTTL](https://redis.io/commands/pttl) | `PTTL key` | Get the time to live for a key in milliseconds |
| [RANDOMKEY](https://redis.io/commands/randomkey) | `RANDOMKEY` | Return a random key from the keyspace |
| [RENAME](https://redis.io/commands/rename) | `RENAME key newkey` | Rename a key |
| [RENAMENX](https://redis.io/commands/renamenx) | `RENAMENX key newkey` | Rename a key, only if the new key does not exist |
| [RESTORE](https://redis.io/commands/restore) | `RESTORE key ttl serialized-value [REPLACE] [ABSTTL] [IDLETIME seconds] [FREQ frequency]` | Create a key using the provided serialized value, previously obtained using DUMP |
| [SCAN](https://redis.io/commands/scan) | `SCAN cursor [MATCH pattern] [COUNT count]` | Incrementally iterate the keys space |
| [SORT](https://redis.io/commands/sort) | `SORT key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC\|DESC] [ALPHA] [STORE destination]` | Sort the elements in a list, set or sorted set |
| [TOUCH](https://redis.io/commands/touch) | `TOUCH key [key ...]` | Alters the last access time of a key(s). Returns the number of existing keys specified |
| [TTL](https://redis.io/commands/ttl) | `TTL key` | Get the time to live for a key |
| [TYPE](https://redis.io/commands/type) | `TYPE key` | Determine the type stored at key | 
| [UNLINK](https://redis.io/commands/unlink) | `UNLINK key [key ...]` | Delete a key asynchronously in another thread. Otherwise it is just as DEL, but non blocking |
| [WAIT](https://redis.io/commands/wait) | `WAIT numreplicas timeout` | Wait for the synchronous replication of all the write commands sent in the context of the current connection | 