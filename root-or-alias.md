让人容易混淆的Nginx的root和alias指令

Nginx的root和alias两个指令之所以让很多人混淆，根本原因还是没有弄明白这两个指令的设计用意和适用场景。现在有这么一个需求：开发小程序时，请求的域名腾讯都需要验证，所以开发经常需要放一个txt验证文件到域名根目录下面，然后腾讯通过对这个文件的访问来进行域名验证。因为历史原因，代码的根目录不对外开放，这就需要单独定义一个location给txt验证文件使用。那这个location怎么写呢，首先看一个写法，你觉得请求 http://test.com/1234.txt Nginx会乖乖返回200么？

```
location /1234.txt {
    alias /data/w3/txt/;
}
```

想好了么？想好了我就公布答案了，Nginx的accesslog里面会有两条记录，一条301，一条403，意不意外，惊不惊喜？具体原因暂时不表，先来看看alias和root的官方文档。
```
[23/Aug/2018:11:47:28 +0800] "GET /1234.txt HTTP/1.1" 301 284
[23/Aug/2018:11:47:28 +0800] "GET /1234.txt/ HTTP/1.1" 403 596
```

## root和alias的官方定义

- root:Sets the root directory for requests. A path to the file is constructed by merely adding a URI to the value of the root directive. 文档说的很清楚，root就是用来定义请求的根目录的，最终访问的实际文件的路径就是root的值+URI，比如下面这种写法，访问 /i/top.gif 的话，实际返回的文件是 /data/w3/i/top.gif。而且文档还特别强调：**如果需要修改URI，请使用alias**。其实root还是比较容易理解。
```
location /i/ {
    root /data/w3;
}
```
- alias:Defines a replacement for the specified location. If alias is used inside a location defined with a regular expression then such regular expression should contain captures and alias should refer to these captures。顾名思义，alias就是一个别名，就是用alias指令所指定的位置的别名。**如果alias用在正则定义的location里面的话，正则里面必须包含捕获组，且alias必须使用这些捕获组**。

这里面容易引起混淆的就是这个别名所代表的具体内容。分两种情况：
#### location定义的path是一个目录
定义如下，访问 /i/top.gif，实际返回文件是 /data/w3/images/top.gif，注意这里对 */i/* 部分用alias的值进行替换了
```
location /i/ {
    alias /data/w3/images/;
}
```
如果location的path跟alias后部分相同，则建议使用root，下面两者等价。
```
location /images/ {
    alias /data/w3/images/;
}

location /images/ {
    root /data/w3;
}
```

#### location定义的path是文件
顾名思义，文件的别名肯定还是文件，所以此时alias必须配置为文件。如下文
```
location /i/top.gif {
    alias /data/w3/images/top.gif;
}
location ~ ^/users/(.+\.(?:gif|jpe?g|png))$ {
    alias /data/w3/images/$1;
}
```

好了，再回到刚开始那个奇怪的例子。仔细分析需求，这是一个 **不需要修改URI** 的需求，适合使用root，写法很简单。如果非要用alias，也能写出来，就是更复杂一些。
```
location ~* ^/.+\.txt$ {
    root /path/to/nginx/root/path/;
}
location ~* ^/(.+\.txt)$ {
    alias /path/to/nginx/root/path/$1;
}
```

现在再来看文头的写法就不难理解了，第一次访问匹配到location，通过alias置换之后，对应的返回文件为/data/w3/txt/，对Nginx来说，当URL指向一个目录并且在最后没有包含"/"时，Nginx 内部会自动的做一个301重定向，并自动添加上"/"，此时请求变成http://test.com/1234.txt/，该访问依旧能进匹配到该location，而/data/w3/txt/index.html并不存在，所以返回403。顺着这个思路走下去，请问访问 http://test.com/1234.txt/1234.txt 会返回200么？答案显然是肯定的。

