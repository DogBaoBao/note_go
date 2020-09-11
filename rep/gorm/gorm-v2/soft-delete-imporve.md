---
description: v1 升级到 v2  + 软删除使用采坑贴
---

# Gorm 软删除使用手册

## 背景

gorm 的软删除默认是 `deleted_at` 字段，值为 NULL，即插入的时候 `deleted_at` 为 NULL，查找的时候带上 `WHERE deleted_at = NULL`。

## 改进

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

运行的测试代码

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

