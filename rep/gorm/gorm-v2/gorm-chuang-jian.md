# Gorm 创建

## 创建对象

### 通过数据指针创建

#### 不带 default（会设置默认值）

```go
type User struct {
	gorm.Model
	Name     string
	Age      uint
	Birthday *time.Time
	Nickname *string
	Address  string
}

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

输出 SQL

```sql
INSERT INTO `gdcloud_user` (`created_at`,`updated_at`,`deleted_at`,`name`,`age`,`birthday`,`nickname`,`address`) VALUES ("2020-09-10 19:25:53.835","2020-09-10 19:25:53.835",NULL,"tiecheng",18,"2020-09-10 19:25:53.835",NULL,"")
```

可以看到  `Nickname` 被设置为了 NULL，因为用了指针 `*string` ，而 `Address` 被设置成了 `""`。

#### default 设置

改变 Address 字段，```gorm:"default:hangzhou"``` 如下：

```go
type User struct {
	gorm.Model
	Name     string
	Age      uint
	Birthday *time.Time
	Nickname *string
	Address  string `gorm:"default:hangzhou"`
}
```

输出 SQL

```sql
INSERT INTO `gdcloud_user` (`created_at`,`updated_at`,`deleted_at`,`name`,`age`,`birthday`,`nickname`,`address`) VALUES ("2020-09-10 19:29:28.612","2020-09-10 19:29:28.612",NULL,"tiecheng",18,"2020-09-10 19:29:28.612",NULL,"hangzhou")
```

`Address` 的插入默认值已经变成了 `"hangzhou"`

#### 高端插入（Upsert）

```go
func TestUpsert(t *testing.T) {
   time := time.Now()
   user := User{
      Name:     "tiecheng",
      Age:      18,
      Birthday: &time,
   }

   DB.Clauses(clause.OnConflict{
      Columns:   []clause.Column{{Name: "name"}, {Name: "nickname"}, {Name: "deleted_at"}},
      DoUpdates: clause.Assignments(map[string]interface{}{"name": user.Name, "nickname": user.Nickname, "deleted_at": nil}),
   }).Create(&user)
}
```

输出 SQL

```sql
INSERT INTO `gdcloud_user` (`created_at`,`updated_at`,`deleted_at`,`name`,`age`,`birthday`,`nickname`,`address`) VALUES ("2020-09-10 19:41:09.159","2020-09-10 19:41:09.159",NULL,"tiecheng",18,"2020-09-10 19:41:09.159",NULL,"hangzhou") ON DUPLICATE KEY UPDATE `deleted_at`=NULL,`name`="tiecheng",`nickname`=NULL
```

