
# table

table 属于 lua 原生提供的模块，对于原生接口使用可以参考：[lua官方文档](https://www.lua.org/manual/5.1/manual.html#5.5)

Xmake 中对其进行了扩展，增加了一些扩展接口：

## table.join

- 合并多个table并返回

可以将多个table里面的元素进行合并后，返回到一个新的table中，例如：

```lua
local newtable = table.join({1, 2, 3}, {4, 5, 6}, {7, 8, 9})
```

结果为：`{1, 2, 3, 4, 5, 6, 7, 8, 9}`

并且它也支持字典的合并：

```lua
local newtable = table.join({a = "a", b = "b"}, {c = "c"}, {d = "d"})
```

结果为：`{a = "a", b = "b", c = "c", d = "d"}`

## table.join2

- 合并多个table到第一个table

类似[table.join](#table-join)，唯一的区别是，合并的结果放置在第一个参数中，例如：

```lua
local t = {0, 9}
table.join2(t, {1, 2, 3})
```

结果为：`t = {0, 9, 1, 2, 3}`

## table.unique

- 对table中的内容进行去重

去重table的元素，一般用于数组table，例如：

```lua
local newtable = table.unique({1, 1, 2, 3, 4, 4, 5})
```

结果为：`{1, 2, 3, 4, 5}`

## table.slice

- 获取table的切片

用于提取数组table的部分元素，例如：

```lua
-- 提取第4个元素后面的所有元素，结果：{4, 5, 6, 7, 8, 9}
table.slice({1, 2, 3, 4, 5, 6, 7, 8, 9}, 4)

-- 提取第4-8个元素，结果：{4, 5, 6, 7, 8}
table.slice({1, 2, 3, 4, 5, 6, 7, 8, 9}, 4, 8)

-- 提取第4-8个元素，间隔步长为2，结果：{4, 6, 8}
table.slice({1, 2, 3, 4, 5, 6, 7, 8, 9}, 4, 8, 2)
```

## table.contains

- 判断 table 中包含指定的值

```lua
if table.contains(t, 1, 2, 3) then
    -- ...
end
```

只要 table 中包含 1, 2, 3 里面任意一个值，则返回 true

## table.orderkeys

- 获取有序的 key 列表

`table.keys(t)` 返回的 key 列表顺序是随机的，想要获取有序 key 列表，可以用这个接口。
