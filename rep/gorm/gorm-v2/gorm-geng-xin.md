# Gorm 更新

```go
	time := time.Now()
	user := User{
		Name:     "tiecheng",
		Age:      19,
		Birthday: &time,
	}

	DB.Model(&user).Where("name = ?", user.Name).Update("nickname", "鐵手")
```

输出 SQL

```sql
UPDATE `gdcloud_user` SET `nickname`="鐵手",`updated_at`=1599739850 WHERE name = "tiecheng"
```



