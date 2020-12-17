#Golang知识点记录
*基础知识可参考*[菜鸟教程](https://www.runoob.com/go/go-tutorial.html)
##一.Go环境安装和IDE的安装设置

##二.Golang学习中常用知识点
###1.关于数据库sql.DB连接的几个参数优化
- SetMaxOpenConns 用于设置最大打开的连接数，默认值为0，表示不限制
- SetMaxIdleConns 用于设置闲置的连接数，默认值为2
- SetConnMaxLifetime 可以限制一个连接使用的最大时长，默认值为0,表示不限制

>SetMaxOpenConns
>> 默认情况下，连接池的最大数量是没有限制的。一般来说，连接数越多，访问数据库的性能越高。但是系统资源不是无限的，数据库的并发能力也不是无限的。因此为了减少系统和数据库崩溃的风险，可以给并发连接数设置一个上限，这个数值一般不超过进程的最大文件句柄打开数，不超过数据库服务自身支持的并发连接数，比如1000。
---
>SetMaxIdleConns
>> 理论上maxIdleConns连接的上限越高，也即允许在连接池中的空闲连接最大值越大，可以有效减少连接创建和销毁的次数，提高程序的性能。但是连接对象也是占用内存资源的，而且如果空闲连接越多，存在于连接池内的时间可能越长。连接在经过一段时间后有可能会变得不可用，而这时连接还在连接池内没有回收的话，后续被征用的时候就会出问题。一般建议maxIdleConns的值为MaxOpenConns的1/2，仅供参考。
---
>SetConnMaxLifetime
>> 设置一个连接被使用的最长时间，即过了一段时间后会被强制回收，理论上这可以有效减少不可用连接出现的概率。当数据库方面也设置了连接的超时时间时，这个值应当不超过数据库的超时参数值

###2.数据库操作优化
>参考[使用go语言操作mysql数据库](https://www.cnblogs.com/tsiangleo/p/4483657.html)

###3.Golang Redis缓存操作
> 初始化
```
package Databases

import (
	"github.com/gomodule/redigo/redis"
)

var RedisPool *redis.Pool
var RedisConn redis.Conn

func RedisPollInit() *redis.Pool {
	return &redis.Pool{
		MaxIdle:   3,
		MaxActive: 5,
		Dial: func() (redis.Conn, error) {
			c, err := redis.Dial("tcp", "127.0.0.1:6379")
			if err != nil {
				return nil, err
			}
			return c, err
		},
	}
}

func RedisInit() {
	RedisPool = RedisPollInit() # 缓存池
	RedisConn = RedisPool.Get() # 缓存句柄,我这里只用了一个，
}

func RedisClose() {
	_ = RedisPool.Close()
}
```
> 基本的使用
```
//在redis操作golang 结构体切片示例
func HandleStructSlice() {
	P := Models.Product{Name: sql.NullString{String: "草", Valid: true}}
	moduleList := []Models.Product{P, P, P, P}
	datas, _ := json.Marshal(moduleList)

	_, _ = RedisConn.Do("SET", "ModuleList", datas) //写入
    _, err = RedisConn.Do("expire", "ModuleList", 60)  //设置过期时间

	rebytes, _ := redis.Bytes(RedisConn.Do("get", "ModuleList"))    //读取
	var object []*Models.Product
	_ = json.Unmarshal(rebytes, &object)
	for _, v := range object {
		fmt.Printf("%+v\n", *v)
	}
}

//在redis操作golang结构体示例
func HandleStruct() {
	var testStruct = &Models.Product{Name: sql.NullString{String: "cx", Valid: true}}
	//json序列化
	datas, _ := json.Marshal(testStruct)
	//缓存数据
	_, _ = RedisConn.Do("set", "struct3", datas)    //写入
    _, err = RedisConn.Do("expire", "struct3", 60)  //设置过期时间

	//读取数据
	rebytes, _ := redis.Bytes(RedisConn.Do("get", "struct3"))   //读取
	//json反序列化
	object := &Models.Product{}
	_ = json.Unmarshal(rebytes, object)
	fmt.Println(object)
}
```

