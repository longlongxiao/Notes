时间：2018/12/3 10:03:12   

参考： 

1. [https://redis.io/commands/eval](https://redis.io/commands/eval)

## Redis Lua 脚本使用

### 脚本简介

**Eval 命令：**

Redis 服务器可以解释执行 Lua脚本，使用脚本可以实现一些复杂的业务逻辑。执行脚本命令如下： 

``` shell
eval("script", key_number, key1, key2, ..., arg1, arg2, arg3,...)
```
**参数含义：**

* `script`: lua脚本。
* `key_number`: key 的数量，当 key_number = 2 时，表示有两个key，其余的参数都是 arg
* `key`: redis 里面数据的key，数量由 key_number 控制。集群模式下相同的key路由到固定的机器。
* `arg`：参数，额外的参数。

**返回值：**

返回值和命令的返回值相同。

**Evalsah 命令：**

类似 eval 相同，唯一的不同是，执行的时候传递的是 `SHA1 签名`，而不是脚本，当缓存中不存在签名对应的脚本时，会返回错误，当缓存中存在则直接执行缓存中的脚本。

**在 Lua 脚本中可有两种方式调用 redis 命令:**

* `redis.call()`: 错误信息会被直接抛出。
* `redis.pcall()`: 错误信息封装到 lua 的 table 数据结构中。

**工具函数：**

```shell
# 响应错误信息
redis.error_reply(error_string) <==> {err = error_string}
# 响应成功信息
redis.status_reply(status_string) <==> {ok = status_string}
# 在Redis服务中打印日志
redis.log(redis.LOG_NOTICE, "message")
```

**支持的函数库：**

* base
* table
* string 
* math
* struct: 打包和解包 lua 结构。
* cjson：json格式化和解析。
* cmsgpack：消息报打包和解包。 
* bitop：位操作。 
* redis.sha1hex：sha1 签名。
* redis.log：写日志信息到 redis log文件，大于等于服务器配置的级别的日志才会写入到日志文件中。
	* redis.LOG_DEBUG
	* redis.LOG_VERBOSE
	* redis.LOG_NOTICE
	* redis.LOG_WARNING
* redis.breakpoint
* redis.debug 

### Redis 和 Lua 数据类型转换  

Redis -> Lua:

	Redis integer reply -> Lua number
	Redis bulk reply -> Lua string
	Redis multi bulk reply -> Lua table (may have other Redis data types nested)
	Redis status reply -> Lua table with a single ok field containing the status
	Redis error reply -> Lua table with a single err field containing the error
	Redis Nil bulk reply and Nil multi bulk reply -> Lua false boolean type

Lua -> Redis:

	Lua number -> Redis integer reply (the number is converted into an integer)
	Lua string -> Redis bulk reply
	Lua table (array) -> Redis multi bulk reply (truncated to the first nil inside the Lua array if any)
	Lua table with a single ok field -> Redis status reply
	Lua table with a single err field -> Redis error reply
	Lua boolean false -> Redis Nil bulk reply.

特殊的类型转换：

1. Lua true  -> redis 1。
2. Lua 的 number 类型会被转换成 Integer，精度会被忽略，因此存储带有精度的数据需要使用 String。
3. Lua 的 array 转换成 Redis 对应数据类型，以数组中的 `nil` 作为结束符号， `nil` 后面的数据会被忽略。

		> eval "return {1,2,3.3333,'foo',nil,'bar'}" 0
		1) (integer) 1
		2) (integer) 2
		3) (integer) 3
		4) "foo"

### 脚本的原子性  

Redis 使用相同的解释器执行命令，脚本会以原子性的方式运行，脚本执行过程中不由其它命令同时执行。在其他客户只能看到脚本执行之前和脚本执行完成之后的状态，不会看到脚本执行的中间状态。


### 其它 

#### 几个命令 

* 刷新(删除)服务器缓存脚本： `script flush`。
* 脚本缓存是否存在：`script exists sha1 sha2 ...`

    ```shell
    SCRIPT EXISTS 7fdd7cbee02972e4c9c018a87d3421260820c9c8 7fdd7cbee02972e4c9c018a87d3421260820c9c8
    1) (integer) 1
    2) (integer) 1
    ```
* 缓存脚本到服务器：`script load script`。
* 终止正在执行的脚本: `script kill`。

#### 脚本内切换数据库

脚本内切换数据库，只会影响脚本执行时操作的数据库，不会影响客户端连接的数据库。

### Java Jedis 客户端  

脚本文件 `hello.lua` 

```lua
local keys = "keys :"
local split_char = ""
for i, v in ipairs(KEYS) do
    keys = keys .. split_char .. "[ " .. i .. " ] = " .. v
    split_char = ";"
end

local args = "args: "
split_char = ";"
for i, v in ipairs(ARGV) do
    args = args .. split_char .. "[ " .. i .. " ] = " .. v
    split_char = ";"
end

redis.log(redis.LOG_NOTICE, keys)
redis.log(redis.LOG_NOTICE, args)

return { keys, args }
```
Java Demo:

```java
private static void executeHello() throws IOException, NoSuchAlgorithmException {
    try (Jedis jedis = jedisPool.getResource()) {
        byte[] script = FileReadUtil.readFromResource("hello.lua");
        String scriptSHA1 = SHA1(script);
        Object response;
        try {
            response = jedis.evalsha(scriptSHA1, 2, "name", "age", "1", "2");
            System.out.println("run sha");
        } catch (Exception e) {
            response = jedis.eval(new String(script), 2, "name", "age", "1", "2");
            System.out.println("run script");
        }
        System.out.println(response);
    }
}

private static String SHA1(byte[] script) throws NoSuchAlgorithmException {
    MessageDigest sha_1 = MessageDigest.getInstance("SHA-1");
    String result = DatatypeConverter.printHexBinary(sha_1.digest(script)).toLowerCase();
    System.out.println(result);
    return result;
}
```
### 脚本调试 

**启动调试脚本：**

```shell
# 逗号前面的是KEYS参数，后面的是ARGV 参数，注意逗号前后各有一个空格
redis-cli --ldb --eval lock.lua book2 , 100 book_num
```
**脚本：**

```lua
--[[参数--]]
local key_name = KEYS[1]
local expire_time = ARGV[1]
local lock_id_key_name = ARGV[2]

--[[初始化锁ID--]]
local exists = redis.call("EXISTS", lock_id_key_name)
if 0 == exists then
    redis.call("SET", lock_id_key_name, 10000)
end

--[[获取锁，获取成功返回锁ID，获取失败返回空字符串--]]
local lock_id = redis.call("INCR", lock_id_key_name)
local set_result = redis.call("SETNX", key_name, lock_id)
if 1 == set_result then
    redis.call("EXPIRE", key_name, expire_time)
    return lock_id .. ''
else
    return ''
end
```
**调试命令：**

	h 查看帮助
	p 输出变量
	s 一步一步执行
	n 下一步
	l 查看代码 
	l 5 3 显示 第五行代码前后三行代码
	w 列出所有代码
	b 查看断点
	b 2 3 在第2、3行添加断点
	b -2 取消第二行的断点
	c 执行到下一个断点
	a 结束调试
	r 执行redis命令 r get name
	e 指定lua命令 e pring
