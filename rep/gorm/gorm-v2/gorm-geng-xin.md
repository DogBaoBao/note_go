# Gorm 更新



```go
func TestUpdate(t *testing.T) {
	time := time.Now()
	user := User{
		Name:     "tiecheng",
		Age:      19,
		Birthday: &time,
	}

	DB.Model(&user).Where("name = ?", user.Name).Update("nickname", "鐵手")
}
```

输出 SQL

```sql
UPDATE `gdcloud_user` SET `nickname`="鐵手",`updated_at`="2020-09-10 20:20:18.147" WHERE name = "tiecheng"
```

如果对象改成

```go
type User struct {
	models.Base
	Name     string
	Age      uint
	Birthday *time.Time
	Nickname *string
	Address  string `gorm:"default:hangzhou"`
}
```

输出 SQL

```sql
UPDATE `gdcloud_user` SET `nickname`="鐵手",`updated_at`=1599739850 WHERE name = "tiecheng"
```

> 一口老血，在 gorm v1 的时候是正常的时间，而 v2 就变成时间戳了

唯一的区别在于 `type:datetime` 这个类型

{% tabs %}
{% tab title="Me" %}
```go
type Base struct {
	ID        int64      `gorm:"primary_key;column:id;type:bigint(20) unsigned;not null;" json:"-"`            // id
	CreatedAt time.Time  `gorm:"column:created_at;type:datetime;not null;" json:"createdAt"`                   // 创建时间
	CreatedBy string     `gorm:"column:created_by;type:varchar(20);not null;default:System;" json:"createdBy"` // 创建人
	UpdatedAt time.Time  `gorm:"column:updated_at;type:datetime;not null;" json:"updatedAt"`                   // 更新时间
	UpdatedBy string     `gorm:"column:updated_by;type:varchar(20);not null;default:System;" json:"updatedBy"` // 更新人
	DeletedAt *time.Time `gorm:"column:deleted_at;type:datetime;not null;" json:"deletedAt"`                   // 删除时间
}
```
{% endtab %}

{% tab title="Gorm" %}
```go
type Model struct {
	ID        uint `gorm:"primarykey"`
	CreatedAt time.Time
	UpdatedAt time.Time
	DeletedAt DeletedAt `gorm:"index"`
}
```
{% endtab %}
{% endtabs %}

去掉这个类型后，输出 SQL 如下：

```sql
UPDATE `gdcloud_user` SET `nickname`="鐵手",`updated_at`="2020-09-10 20:23:46.47" WHERE name = "tiecheng"
```



