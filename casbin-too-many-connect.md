# Casbin 导致的数据库连接满

## 现状

开发环境运行，每次页面刷新都会请求后端服务查询用户信息（没有加缓存，每次查询数据库），突然开始数据库报错

```text
too Many Connections
```

##  排查

数据库连接池满，首先去查看数据库的连接池配置和项目中数据库连接池的配置。

### 数据库信息

**数据库连接池**

```sql
show variables like '%max_connections%';

max_connections	128
```

> 如果要设置数据库连接的最大值呢？

```sql
// 设置最大连接数 1000，但是如果数据库重启，就会失效，要永久有效需要配置到 MySQL 的配置文件
set GLOBAL max_connections=1000; 
```

**查看连接进程**

```sql
show processlist; 

2313	root	localhost:50629		Sleep	53111
2313	root	localhost:60621		Sleep	33149
2313	root	localhost:50893		Sleep	22447
...
2313	root	localhost:50609		Sleep	13143		
2314	root	localhost:50610	casbin	Sleep	13143	
...	
2315	root	localhost:56672	gdcloud	Query	0	starting	show processlist
```

可以看到连接有一大波的进程是处于 Sleep 状态，MySQL 被建立了很多连接但是没有使用

### 项目 Gorm 数据库配置

```go
func gormSetting(db *gorm.DB) {
	// 不需要给表加S
	db.SingularTable(true)
	// 连接池配置
	db.DB().SetMaxIdleConns(10)
	db.DB().SetMaxOpenConns(100)
	db.DB().SetConnMaxLifetime(time.Hour)
	// 打开日志
	db.LogMode(true)
	db.Callback().Create().Replace("gorm:update_time_stamp", updateTimeStampForCreateCallback)
	// 设置表名前缀
	gorm.DefaultTableNameHandler = func(db *gorm.DB, defaultTableName string) string {
		return "gdcloud_" + defaultTableName
	}
}
```

* 最大打开连接数 100
* 最大闲置连接数 10
* 连接最大保持时间 1小时

但是 MySQL 的 process 是大于 100的，并且观察请求日志因为涉及到权限，每次请求都会走 casbin 的代码访问，因为项目中除了 gorm 外就 casbin 和数据库有交互。

###  Casbin 持久化逻辑

> 代码从其它列子抄来的，未曾琢磨

```go
// Add casbin policy.
func AddPolicy(p permissionModel) (bool, error) {
	if e, err := Casbin(); err != nil {
		return false, err
	} else {
		return e.AddPolicy(p.V0, p.V1, p.V2)
	}
}

// Persisten to db.
// %s:%s@(%s:%s)/%s?charset=utf8&parseTime=True&loc=Local
func Casbin() (c *casbin.Enforcer, err error) {
	if a, err := gormadapter.NewAdapter("mysql", "root:root@(127.0.0.1:3306)/"); err != nil {
		return nil, err
	} else {
		if e, err := casbin.NewEnforcer("pkg/config/rbac_model.conf", a, true); err != nil {
			return nil, err
		} else {
			e.LoadPolicy()
			return e, nil
		}
	}
}
```

```go
func AuthCheckRole(c *gin.Context) {
	//根据上下文获取 claims 从 claims 获得 roles
	claims := c.MustGet("claims").(*jwt.Customclaims)
	role := claims.Roles
	if e, err := casbin.Casbin(); err != nil {
	......
	}
	
	......
}
```

看了下代码，就猜到是 Casbin 的问题了，因为每次页面请求都会验证角色权限，而每次都会调用 `casbin.Casbin()`，它又是调用到了 `gormadapter.NewAdapter()` ，每一次都会 New 的话就很可能占用了一次连接。继续跟进去：

```go
func NewAdapter(driverName string, dataSourceName string, dbSpecified ...bool) (*Adapter, error) {
	a := &Adapter{}
	a.driverName = driverName
	a.dataSourceName = dataSourceName

	......

	// Open the DB, create it if not existed.
	err := a.open()
	
	......

	return a, nil
}
```

可以看到这里有 `a.open()` 操作，写着打开一个 DB，点进去看：

```go
func (a *Adapter) open() error {
	var err error
	var db *gorm.DB

	if a.dbSpecified {
		db, err = openDBConnection(a.driverName, a.dataSourceName)
		if err != nil {
			return err
		}
	} else {
		if err = a.createDatabase(); err != nil {
			return err
		}
		if a.driverName == "postgres" {
			db, err = openDBConnection(a.driverName, a.dataSourceName+" dbname=casbin")
		} else if a.driverName == "sqlite3" {
			db, err = openDBConnection(a.driverName, a.dataSourceName)
		} else {
			db, err = openDBConnection(a.driverName, a.dataSourceName+"casbin")
		}
		if err != nil {
			return err
		}
	}
	a.db = db
	return a.createTable()
}
```

看到这里的 `openDBConnection()` 就彻底明白了，`*gorm.DB` 对象每一次都会创建，每一次创建都会建立连接，最终导致 MySQL 连接满。

## 改进

每次连接改成创建的对象的时候创建一次连接，后面共用 `*gorm.DB`。

```go
type Permission struct {
	adapter    *gormadapter.Adapter
	enforcer   *casbin.Enforcer
	configPath string
}

func NewPermission(dbModel config.DbConfig, path string) *Permission {
	p := &Permission{
		configPath: path, // "pkg/config/rbac_model.conf"
	}
	connArgs := fmt.Sprintf("%s:%s@(%s:%s)/", dbModel.Username, dbModel.Password, dbModel.Host, dbModel.Port)

	if a, err := gormadapter.NewAdapter("mysql", connArgs); err != nil {
		panic(err)
	} else {
		p.adapter = a

		if e, err := casbin.NewEnforcer(p.configPath, p.adapter, true); err != nil {
			panic(err)
		} else {
			p.enforcer = e
		}
	}

	return p
}

func GetPermission() *Permission {
	onceNew.Do(func() {
		PermissionService = NewPermission(config.GetMustConfig().Orgrimmar.DbConfig, config.GetMustConfig().Orgrimmar.CasbinPath)
	})

	return PermissionService
}

// 增加权限
func (p *Permission) AddPermissionParams(params ...string) (bool, error) {
	var pm PermissionModel
	l := len(params)
	switch {
	case l == 4 && params[0] == "p":
		pm = p.make3PermissionModel("p", params[0], params[1], params[2])
	case l == 3 && params[0] == "g":
		pm = p.make2PermissionModel("g", params[0], params[1])
	default:
		return false, nil
	}

	return p.AddPermission(pm)
}

// Check 验证权限
func (p *Permission) Check(p1, p2, p3 string) (bool, error) {
	return p.enforcer.Enforce(p1, p2, p3)
}

// RemovePolicy 删除策略 传入 p 对应的 v0 v1 v2 参数
func (p *Permission) RemovePolicy(params ...string) (bool, error) {
	return p.enforcer.RemovePolicy(params)
}

......
```

因为暂时配置目前不需要动态刷新，所以直接共享了下面两个变量：

```text
adapter    *gormadapter.Adapter
enforcer   *casbin.Enforcer
```

后面对 casbin 的操作都基于了 enforcer。

