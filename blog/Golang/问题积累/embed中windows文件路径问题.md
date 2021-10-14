## embed在windows中文件路径报file does not exist
2021年10月15日
### 问题描述
一个需要通过go 1.16+版本的embed包获取文件的功能，在windows中报文件不存在，但实际上是存在的。
### 问题代码
```go

//go:embed static
var staticDir

func getXXX(subPath string) string {
    path = filepath.Join("static", subPath)
	fs, err := staticDir.ReadDir(path)
    if err != nil {
        // 这里会报`static\img`: file does not exist
        // TODO
    }
    return ""
}

func main() {
    getXXX("img")
}
```
### 思考

实际上目录是有的，那么看着就是反斜线的问题，手动输入正斜线例子测试，确实是正确的，所以接下来考虑如何在windows下，将多个路径通过正斜线拼接。

### 探索

`filepath.Join()`返回的值，是反斜杠`\`作为路径分割符的（static\img）。那么替换成斜线就好了。

1.可以用`path.Join()`，替换掉`filepath.Join()`，前者直接用斜线，后者是根据os自动更改的。

> Join joins any number of path elements into a single path, separating them with `slashes`. Empty elements are ignored. The result is Cleaned. However, if the argument list is empty or all its elements are empty, Join returns an empty string.

2.`filepath.ToSlash()`二次处理`filepath.Join()`结果，前者可以将路径中的分割符变成正斜线。

### 待解决

1.embed包中ReadDir方法，为什么不支持反斜线。

看了一下，`embed.FS`获取目录下面的各个子目录&文件，路径分割符就是用的正斜线，然后在`ReadDir()`方法中有一步，判断路径是否存在的逻辑：

```go
// Go/src/embed/embed.go:269
if i < len(files) && trimSlash(files[i].name) == name{
	return &files[i]
}
return nil
```

files中的目录始终都是正斜线，而`name`是反斜线，所以不会相等。

### 参考

1. [filepath.Join returning path as \ instead of / on WINDOWS · Issue #30616 · golang/go (github.com)](https://github.com/golang/go/issues/30616)
2. [path package - path - pkg.go.dev](https://pkg.go.dev/path#Join)

