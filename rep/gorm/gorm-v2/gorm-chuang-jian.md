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

## 项目实战

### 主键回填失败

![](../../../.gitbook/assets/image%20%282%29.png)

> | primaryKey | 指定列为主键 |
> | :--- | :--- |

修改 Base 

```go
type Base struct {
	ID        int64          `gorm:"primarykey;column:id;type:bigint(20) unsigned;not null;autoIncrement" json:"-"` // id
	CreatedAt time.Time      `gorm:"column:created_at;not null;" json:"createdAt"`                                  // 创建时间
	CreatedBy string         `gorm:"column:created_by;type:varchar(20);not null;default:System;" json:"createdBy"`  // 创建人
	UpdatedAt time.Time      `gorm:"column:updated_at;type:datetime;not null;" json:"updatedAt"`                    // 更新时间
	UpdatedBy string         `gorm:"column:updated_by;type:varchar(20);not null;default:System;" json:"updatedBy"`  // 更新人
	DeletedAt gorm.DeletedAt `gorm:"column:deleted_at;not null;" json:"deletedAt"`                                  // 删除时间
}
```

### 输出 SQL打印对不上

```sql
INSERT INTO `gdcloud_user` (`id`,`created_at`,`created_by`,`updated_at`,`updated_by`,`deleted_at`,`name`,`nickname`,`chinese_name`,`english_name`,`first_name`,`second_name`,`ali_staff_id`,`phone`,`email`,`second_email`,`dept_id`,`position_id`,`entry_at`,`hometown`,`head_img_url`,`status`) VALUES (0,"2020-09-11 09:58:00.486","System","2020-09-11 09:58:00.486","System","1987-06-05 04:03:02","tiecheng.tc","鐵手","铁城",NULL,NULL,NULL,NULL,"18768171164","tiecheng.tc@gongdao.com",NULL,0,0,"2020-09-11 09:58:00.486",NULL,"https://avatars2.githubusercontent.com/u/553434601s=400&u=8f60cf1a440ea3395401df0e0e880c9d37aae395&v=4",?)
```



