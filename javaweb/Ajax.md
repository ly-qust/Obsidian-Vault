1.同步和异步
同步是客户端向服务器端发送请求时需要等待，异步则不需要，可以去执行其他事情
2.Axios是对原生的AJAX进行封装，简化书写
https://www.axios-http.cn
3.
引入axios文件
```HTML
<script src="https://unpkg.com/axios/dist/axios.min.js"></script>
```
get请求
```javaScript
axios.get("https://mock.apifox.cn/m1/3083103-0-default/emps/list").then(result => {
    console.log(result.data);
})
```
post请求
```JavaScript
axios.post("https://mock.apifox.cn/m1/3083103-0-default/emps/update","id=1").then(result => {
    console.log(result.data);
})
```
4.
## **then 之前的内容是异步 → 不等待**

## **但 then 回调内部的代码是同步 → 按顺序执行**
```javascript
内部同步
.then(result => {
    console.log(result.data); // 一定最先（因为你写在第一行）
    // 再执行你的一大堆代码
})

外部异步
console.log("1. start");

axios.get("url").then(result => {
    console.log("3. 结果回来了");
});

console.log("2. end");
1. start
2. end
3. 结果回来了
```


