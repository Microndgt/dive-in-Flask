StringIO
===

数据读写可以在内存中进行，在内存中读写str，要把str写入StringIO，我们需要先创建一个StringIO，然后，像文件一样写入

```
from io import StringIO
f = StringIO()
# 将会返回写入的数目
f.write("hello")
# 获取已经写入的str数据
f.getvalue()
```

可以用str初始化StringIO，然后，和文件一样读取，有read,readline,seek方法

```
g = StringIO("i love u")
for i in g.read():
    print(i)
```

BytesIO
===

StringIO操作的只能是str，如果要操作二进制数据，就需要使用BytesIO。

BytesIO实现了在内存中读写bytes

```
from io import BytesIO
h = BytesIO()
# 必须传入编码后的数据，二进制数据
h.write('s'.encode('utf-8'))
h.getvalue()
```

和StringIO类似，可以用一个bytes初始化BytesIO，然后，像读文件一样读取：

```
i = BytesIO(b'\xe4\xb8\xad')
i.read().decode('utf-8')
```
