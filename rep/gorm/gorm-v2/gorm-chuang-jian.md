# Gorm 创建

## 创建对象

### 字段默认值填充

#### 指针对象和非指针对象

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

