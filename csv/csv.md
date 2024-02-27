## CSV 文件处理





### 常见问题

#### record on line 2: wrong number of fields

解决：设置 `FieldsPerRecord = -1`

示例

```go
csvFile, _ := os.Open("file.csv")
r := csv.NewReader(bufio.NewReader(csvFile))
r.Comma = ';'
// 不检查每一行的字段数量
r.FieldsPerRecord = -1
```



#### parse error on line 1, column 4: bare " in non-quoted-field

解决：设置 `LazyQuotes = true`

示例

```go
fd, _ := os.Open("file.csv")
r := csv.NewReader(bufio.NewReader(fd))
r.Comma = ';'
r.LazyQuotes = true
```

