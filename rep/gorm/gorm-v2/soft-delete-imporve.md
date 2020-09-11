---
description: v1 升级到 v2  + 软删除使用采坑贴
---

# Gorm 软删除使用手册

## 背景

gorm 的软删除默认是 `deleted_at` 字段，值为 NULL，即插入的时候 `deleted_at` 为 NULL，查找的时候带上 `WHERE deleted_at = NULL`。

## 改进

### 插入的时候加默认值

插入的时候自动填充 `deleted_at` 字段为默认字段

```go
type Base struct {
	ID        int64      `gorm:"primary_key;column:id;type:bigint(20) unsigned;not null;" json:"-"`            // id
	CreatedAt time.Time  `gorm:"column:created_at;not null;" json:"createdAt"`                                 // 创建时间
	CreatedBy string     `gorm:"column:created_by;type:varchar(20);not null;default:System;" json:"createdBy"` // 创建人
	UpdatedAt time.Time  `gorm:"column:updated_at;type:datetime;not null;" json:"updatedAt"`                   // 更新时间
	UpdatedBy string     `gorm:"column:updated_by;type:varchar(20);not null;default:System;" json:"updatedBy"` // 更新人
	DeletedAt *time.Time `gorm:"column:deleted_at;not null;" json:"deletedAt"`                                 // 删除时间
}

func (b *Base) BeforeCreate(tx *gorm.DB) (err error) {
	if b.DeletedAt == nil {
		b.DeletedAt = &db.DEFAULT_TIME
	}
	return
}
```

通过 `BeforeCreate` 方法在查询的时候自动添加 `deleted_at` 为默认字段。

运行插入测试代码：

```go
func TestCreate(t *testing.T) {
	time := time.Now()
	user := User{
		Name:     "tiecheng",
		Age:      18,
		Birthday: &time,
	}

	DB.Create(&user)
}
```

输出 SQL：

```sql
INSERT INTO `gdcloud_user` (`id`,`created_at`,`created_by`,`updated_at`,`updated_by`,`deleted_at`,`name`,`age`,`birthday`,`nickname`,`address`) VALUES (0,"2020-09-11 16:39:36.211","System","2020-09-11 16:39:36.211","System","1987-06-05 04:03:02","tiecheng",18,"2020-09-11 16:39:36.21",NULL,"hangzhou")
```

### 删除的时候软删除

运行删除测试代码：

```go
func TestDelete(t *testing.T) {
	DB.Where("name = ?", "tiecheng").Delete(&User{})
}
```

发现并不是软删除，输出 SQL 如下：

```sql
DELETE FROM `gdcloud_user` WHERE name = "tiecheng"
```

查看了官方的 `gorm.Model` 对象，在 v1是只需要有这个字段，v2 升级后，变成了如下对象：

```go
type DeletedAt sql.NullTime
```

于是基本对象和创建的时候设置时间的代码改为：

```go
type Base struct {
	ID        int64          `gorm:"primary_key;column:id;type:bigint(20) unsigned;not null;" json:"-"`            // id
	CreatedAt time.Time      `gorm:"column:created_at;not null;" json:"createdAt"`                                 // 创建时间
	CreatedBy string         `gorm:"column:created_by;type:varchar(20);not null;default:System;" json:"createdBy"` // 创建人
	UpdatedAt time.Time      `gorm:"column:updated_at;type:datetime;not null;" json:"updatedAt"`                   // 更新时间
	UpdatedBy string         `gorm:"column:updated_by;type:varchar(20);not null;default:System;" json:"updatedBy"` // 更新人
	DeletedAt gorm.DeletedAt `gorm:"column:deleted_at;not null;" json:"deletedAt"`                                 // 删除时间
}

func (b *Base) BeforeCreate(tx *gorm.DB) (err error) {
	if !b.DeletedAt.Valid {
		b.DeletedAt = gorm.DeletedAt(sql.NullTime{Time: db.DEFAULT_TIME, Valid: true})
	}
	return
}
```

再运行删除，输出 SQL

```sql
UPDATE `gdcloud_user` SET `deleted_at`="2020-09-11 12:52:21.529" WHERE name = "tiecheng"
```

### 查找的时候带默认删除时间

> 核心就是修改下图红色的值

![](../../../.gitbook/assets/image%20%281%29.png)

暴力实现：

```go
db.Callback().Query().Before("gorm:query").Register("gorm:software_condition", softwareCondition)
```

```go
func softwareCondition(db *gorm.DB) {
	db.Statement.Schema.QueryClauses = QueryClauses()
}

func QueryClauses() []clause.Interface {
	return []clause.Interface{
		clause.Where{Exprs: []clause.Expression{
			clause.Eq{
				Column: clause.Column{Table: clause.CurrentTable, Name: "deleted_at"},
				Value:  gorm.DeletedAt(sql.NullTime{Time: DEFAULT_TIME, Valid: true}),
			},
		}},
	}
}
```

在进行 `query` 这个 `callback` 之前修改 `db.Statement.Schema.QueryClauses` 里面 `deleted_at` 的 `value` 值为业务需要的默认删除时间即可，我这里为了简单就直接替换了整个数组。

## 源码分析

### 有哪些官方自带的callback

```go
func RegisterDefaultCallbacks(db *gorm.DB, config *Config) {
	enableTransaction := func(db *gorm.DB) bool {
		return !db.SkipDefaultTransaction
	}

	createCallback := db.Callback().Create()
	createCallback.Match(enableTransaction).Register("gorm:begin_transaction", BeginTransaction)
	createCallback.Register("gorm:before_create", BeforeCreate)
	createCallback.Register("gorm:save_before_associations", SaveBeforeAssociations)
	createCallback.Register("gorm:create", Create(config))
	createCallback.Register("gorm:save_after_associations", SaveAfterAssociations)
	createCallback.Register("gorm:after_create", AfterCreate)
	createCallback.Match(enableTransaction).Register("gorm:commit_or_rollback_transaction", CommitOrRollbackTransaction)

	queryCallback := db.Callback().Query()
	queryCallback.Register("gorm:query", Query)
	queryCallback.Register("gorm:preload", Preload)
	queryCallback.Register("gorm:after_query", AfterQuery)

	deleteCallback := db.Callback().Delete()
	deleteCallback.Match(enableTransaction).Register("gorm:begin_transaction", BeginTransaction)
	deleteCallback.Register("gorm:before_delete", BeforeDelete)
	deleteCallback.Register("gorm:delete", Delete)
	deleteCallback.Register("gorm:after_delete", AfterDelete)
	deleteCallback.Match(enableTransaction).Register("gorm:commit_or_rollback_transaction", CommitOrRollbackTransaction)

	updateCallback := db.Callback().Update()
	updateCallback.Match(enableTransaction).Register("gorm:begin_transaction", BeginTransaction)
	updateCallback.Register("gorm:setup_reflect_value", SetupUpdateReflectValue)
	updateCallback.Register("gorm:before_update", BeforeUpdate)
	updateCallback.Register("gorm:save_before_associations", SaveBeforeAssociations)
	updateCallback.Register("gorm:update", Update)
	updateCallback.Register("gorm:save_after_associations", SaveAfterAssociations)
	updateCallback.Register("gorm:after_update", AfterUpdate)
	updateCallback.Match(enableTransaction).Register("gorm:commit_or_rollback_transaction", CommitOrRollbackTransaction)

	db.Callback().Row().Register("gorm:raw", RowQuery)
	db.Callback().Raw().Register("gorm:raw", RawExec)
}
```

上文中添加自定义 `callback` 也是根据这个默认的来加，通过 `Before()` 或 `After()` 控制顺序。

### 如何设置的删除参数

```go
func (DeletedAt) QueryClauses() []clause.Interface {
	return []clause.Interface{
		clause.Where{Exprs: []clause.Expression{
			clause.Eq{
				Column: clause.Column{Table: clause.CurrentTable, Name: "deleted_at"},
				Value:  nil,
			},
		}},
	}
}
```

因为这里实现了 `QueryClauses() []clause.Interface` 方法，所以实现了:

```go
type QueryClausesInterface interface {
	QueryClauses() []clause.Interface
}
```

### 哪里设置

```go
func (schema *Schema) ParseField(fieldStruct reflect.StructField) *Field {
  ......
    
	fieldValue := reflect.New(field.IndirectFieldType)

	if fc, ok := fieldValue.Interface().(CreateClausesInterface); ok {
		field.Schema.CreateClauses = append(field.Schema.CreateClauses, fc.CreateClauses()...)
	}

	if fc, ok := fieldValue.Interface().(QueryClausesInterface); ok {
		field.Schema.QueryClauses = append(field.Schema.QueryClauses, fc.QueryClauses()...)
	}

	if fc, ok := fieldValue.Interface().(UpdateClausesInterface); ok {
		field.Schema.UpdateClauses = append(field.Schema.UpdateClauses, fc.UpdateClauses()...)
	}

	if fc, ok := fieldValue.Interface().(DeleteClausesInterface); ok {
		field.Schema.DeleteClauses = append(field.Schema.DeleteClauses, fc.DeleteClauses()...)
	}    
  
  ......
}
```

可以看到四种类型的 Clause 都是在这里设置的，通过对字段进行断言。

### 哪里拼接

```go
func (eq Eq) Build(builder Builder) {
	builder.WriteQuoted(eq.Column)

	if eq.Value == nil {
		builder.WriteString(" IS NULL")
	} else {
		builder.WriteString(" = ")
		builder.AddVar(builder, eq.Value)
	}
}
```

