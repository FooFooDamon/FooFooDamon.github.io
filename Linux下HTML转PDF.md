<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# Linux下HTML转PDF.md

## htmldoc

### 用法

待补充

### 示例

```
htmldoc -t pdf13 --webpage -f xx.pdf *.html
```

## pandoc

### 用法

待补充

### 示例

```
pandoc -o xx.pdf *.html
```

## wkhtmltopdf

### 用法

待补充

### 示例

本地文档：
```
wkhtmltopdf --no-images --disable-javascript --disable-local-file-access --disable-plugins xx.html xx.pdf
```

网址：
```
wkhtmltopdf 'https://github.com/' github.pdf
```

## pdftk（用于合并多个PDF）

### 用法

待补充

### 示例

```
pdftk *.pdf output xx.pdf
```

## 参考

[Ubuntu下只需 3 步将 chm 转为 pdf](https://www.linuxidc.com/Linux/2012-11/74268.htm)

[在Ubuntu下如何将chm文件转成pdf格式的方法介绍](https://www.aliyun.com/jiaocheng/177204.html)

