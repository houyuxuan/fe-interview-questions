# fe-interview-questions 2022.2.11
前端面试题总结
## 1.http 缓存机制
### 协商缓存
#### no-cache
当 header（包括请求头和响应头）中 Cache-Control（HTTP/1.1规范）或 Pragma（HTTP/1.0规范）字段包含`no-cache`指令时，触发协商缓存。

- ETag 和 If-None-Match

ETag 是一个 URL 资源的标识符（版本号），属性值是一个 hash 值或一个数字版本号，在请求资源服务器返回时会将 ETag 加到 response-header 里面：

```
ETag: "298asdf89wrlkfjc03e848ru5nf8rjfnvurjke"
```
客户端拿到后会保存起来，下次请求时，将 If-None-Match 添加在 request-header 里：

```
If-None-Match: "298asdf89wrlkfjc03e848ru5nf8rjfnvurjke"
```
然后服务端拿到请求里面的`If-None-Match`跟当前版本的`ETag`比较下：

1. 如果是一样的话，直接返回`304`，语义为`Not Modified`，不返回内容(`body`)，只返回`header`，告诉浏览器直接用缓存。
2. 如果不一样的话，返回`200`和最新的内容

- Last-Modified 和 If-Modified-Since

与ETag 和 If-None-Match类似，这两个属性也是配套使用的，只不过属性值为时间。服务端将 Last-Modified随资源返回的 response-header 中：
```
Last-Modified: Fri Feb 11 2022 08:00:00 GMT+0800
```
客户端在使用时，将 If-Modified-Since 放到 request-header 中：
```
If-Modified-Since: Fri Feb 11 2022 08:00:00 GMT+0800
```
服务端拿到这个头后，会跟当前版本的修改时间进行比较：

1. 当前版本的修改时间比这个晚，也就是这个时间后又改过了，返回`200`和新的内容
2. 当前版本的修改时间和这个一样，也就是没有更新，返回`304`，不返回内容，只返回头，客户端直接使用缓存

与`If-Modified-Since`对应的还有`If-Unmodified-Since`，`If-Modified-Since`可以理解为**有更新才下载**，那`If-Unmodified-Since`就是**没有更新才下载**。如果客户端传了`If-Unmodified-Since`，像这样：

```
If-Unmodified-Since: Fri Feb 11 2022 08:00:00 GMT+0800 
```

服务端拿到这个头后，也会跟当前版本的修改时间进行比较：

1. 如果这个时间后没有更新，服务器返回`200`，并返回内容。
2. 如果这个时间后有更新，其实就是这个`if`不成立，会返回错误代码`412`，语义为`Precondition Failed`

- ETag 和 Last-Modified 比较

Last-Modified 的时间精度是秒级，在1秒内多次修改，使用 Last-Modified 上无法准确获取到最新资源。而 ETag 在每次文件更新后都会生成新的值，所以两者对比，ETag 优先级要比 Last-Modified 高。

### 强制缓存
- Expries
在 response-header 中：
```
Expires: Fri Feb 11 2022 08:00:00 GMT+0800
```
在这个时间之前，客户端都不会再对这个资源发起新的请求，直接使用缓存。

- cache-control：max-age （优先级更高）
cache-control 中的 max-age 指令与 expries 类似，expries 设置 的是到期时间节点，max-age 设置的时到期时长：
```
Cache-Control: max-age=30000
```
- cache-control：immutable
``immutable``表示一直使用缓存，再也不用发起请求。但对于这个指令支持的浏览器比较少


### 强制缓存与协商缓存优先级

如果强制缓存被触发，就不会请求到服务器，而协商缓存必须需要请求到服务器核对 ETag 或 Last-Modified 内容，所以强制缓存的优先级要高于协商缓存。当强制缓存时效时，才会触发协商缓存，请求与服务器协商后决定是否使用缓存。

### Cache-Control 其他指令
#### 可缓存性
- `public` 表明响应可以被任何对象（包括：发送请求的客户端，代理服务器，等等）缓存，即使是通常不可缓存的内容。（例如：1.该响应没有`max-age`指令或`Expires`消息头；2. 该响应对应的请求方法是POST。

-   `private` 表明响应只能被单个用户缓存，不能作为共享缓存（即代理服务器不能缓存它）。私有缓存可以缓存响应内容，比如：对应用户的本地浏览器。

- `no-store` 缓存不应存储有关客户端请求或服务器响应的任何内容，即不使用任何缓存。

- `no-cache` 在发布缓存副本之前，强制要求缓存把请求提交给原始服务器进行验证(协商缓存验证)。

#### 到期
- `max-age=<seconds>` 设置缓存存储的最大周期，超过这个时间缓存被认为过期(单位秒)。与`Expires`相反，时间是相对于请求的时间。

- `s-maxage=<seconds>` 覆盖`max-age`或者`Expires`头，但是仅适用于共享缓存(比如各个代理)，私有缓存会忽略它。

- `min-fresh=<seconds>` 表示客户端希望获取一个能在指定的秒数内保持其最新状态的响应。

#### 重新加载和重新验证
- `must-revalidate`  一旦资源过期（比如已经超过`max-age`），在成功向原始服务器验证之前，缓存不能用该资源响应后续请求。
- `proxy-revalidate` 与must-revalidate作用相同，但它仅适用于共享缓存（例如代理），并被私有缓存忽略。
