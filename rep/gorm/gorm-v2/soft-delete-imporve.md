# Gorm 软删除改进

## 背景

gorm 的软删除默认是 `deleted_at` 字段，值为 NULL，即插入的时候 `deleted_at` 为 NULL，查找的时候带上 `WHERE deleted_at = NULL`。

## 改进

插入的时候自动填充 deleted\_at 字段为默认字段

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

查询的时候自动添加 deleted\_at 为默认字段

