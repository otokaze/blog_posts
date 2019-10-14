#### 前言
`Swarmpit` 绝对是我用过在不管是 `PC` 还是 `Mobile` 都有良好支持的 Swarm UI Management，它不但界面漂亮操作起来也很简单应手，最重要的是它是辣么的轻便！如果你没有用过 `Swarmpit` 的话，我再次强烈安利给大家～（这是波无偿彩虹屁^^）

当然 `Swarmpit` 也有美中不足的地方：默认不支持 `https` 部署，再加上与已存在的 `http(s)` 默认端口冲突 ，因此我只能选择通过 `Nginx` 将流量反代到 `Swarmpit`，以此做到端口复用以及 `ssl` 配置。 （毕竟谁都不想在生产环境的域名后面输入一串烦人的 `Port` :(

但也以此引发了下面一系列问题。。。

#### 现象
通过 `Nginx` 反代后发现打开 `Swarmpit` 主页的速度极其的慢，打开 `Console` 观察所发送请求都处于 `Pending` 状态（如图）
![image](https://www.otokaze.cn/wp-content/uploads/2019/10/TCP_handshake.png)
直觉告诉我，应该是浏览器与 `Nginx` 之间建立的 `keeplive` 连接的缓存区一直有数据在处理，从而引起了之后这条 `keeplive` 上的所有请求都发生了阻塞。

为了证实我的猜想，我就去康康 `netstat` 它是怎么说的：
![image](http://ws1.sinaimg.cn/large/c2f00e48gy1g7sivmy7j6j213o0ownky.jpg)
端口 `6080` 也就是我部署的 `Swarmpit` 服务所监听的真实端口，可以看到作为服务方的  `::1:6080` 出现了很多 `tcp6` 不寻常的 `CLOSE_WAIT`，它对应的客户端的地址 `::1:45822` 应该是 `Nginx` 在这里作为反代的客户端通信端口。

这里我就很奇怪，我本机根本没有 `ipv6` 的地址，而且用作反代的 `Nginx` 也没监听 tcpv6 的端口啊，哪来的 `tcpv6` 的连接呢？我再仔细翻阅 `Nginx` 的配置这才发现了其中的端倪：
```nginx
upstream internal_docker_swarm  {
    server localhost:6080;
}
```
我反代的源地址这里用了 `localhost`，我恍然大悟！这在 `/etc/hosts` 里 `localhost` 一般都会同时映射 `ipv4&ipv6` 地址，也就是说这里也是双栈工作的，所以出现 `tcpv6` 的连接也不奇怪了。

那现在我们绕回 `CLOSE_WAIT` 的问题，大家也应该都知道出现 `CLOSE_WAIT` 的情况无非就是客户端主动向服务端提出关闭，但是在服务端还没正常发送 `FIN` 确认关闭状态时客户端的所在 `TCP` 连接已经被强制关闭了，这会导致服务端还是会重发 `ACK` 直到客户端响应，然并卵，你永远都无法唤醒一个已经不存在的连接，等待它的只有系统回收。实在听不明白可以看下图，会让你又更直观的理解。

![image](http://ww1.sinaimg.cn/large/c2f00e48gy1g7tlosz3y9j20kc0fnjsh.jpg)

道理都懂，那到底是为什么会造成这种现象？顺藤摸瓜，我们再仔细看了下浏览器发出的请求，其中的一条 `eventsource` 的请求引起了我的注意（`eventsource` 是个单通道协议，这里客户端仅做 `sub` 角色，只用作做接收服务端发送的数据，协议明细这里就不赘述了，有兴趣的自行 `Google` :），顺带搜了一圈资料，发现 `Nginx` 默认不支持 `eventsource` 协议的反代，就算反代过去服务端也不会认，但也不会报错关闭连接，连接会一直持续到超时为止，不过那时候 `Nginx` 反代过去的连接是否还存在就很随缘了。。。

#### 解决方案
知道了原因，那解决问题就很简单了(ง •_•)ง。
照搬网上的解决方案，修改 `Nginx` 配置如下，使其支持 `eventsource`，那连接就不会被阻塞了：
```nginx
location /events {
    proxy_pass http://internal_docker_swarm;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_buffering off;
    proxy_cache off;
}
```
以防万一，我们把 `tcpv6` 的入口也给关上，反正留着也没卵用，不给喜欢钻空子的 `CLOSE_WAIT` 留下机会半点机会(～￣▽￣)～
```shell
$ echo 'net.ipv6.conf.all.disable_ipv6=1' >> /etc/sysctl.conf
$ sysctl -p
```
大功告成~ 重启后观察了一阵子，效果拔群，自此我就再也没有见到过那惹人厌的 `CLOSE_WAIT` 了。
