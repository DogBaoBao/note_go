# 巧妙的 for 循环

### for 循环越界

#### 情况

for 循环在处理数组和 slice 有区别，所以 for 语句本身的 copy 会导致一些莫名其妙的问题

#### 如何避免

不要用拷贝出来的值

#### 建议

如果 slice 里面的值不是指针，用

```go
for idx := range xxx {
   item = xxx[idx]
}
```

如果里面的值是指针，用

```go
for _,item := range xxx {
   
}
```

#### 实际效果

```go
func BuildChildMenu(menus []Menu) {
	menuMap := make(map[int64]*Menu, len(menus))

	for i := range menus {
		menu := menus[i]
		menuMap[menu.ID] = &menu
	}
	
}
```

menuMap 里面是正常的值

如果是：

```go
func BuildChildMenu(menus []Menu) {
		menuMap := make(map[int64]*Menu, len(menus))

	for _, m := range menus {
		menuMap[m.ParentID] = &m
	}
}	
```

所有的 `*Menu` 值都是一样的

