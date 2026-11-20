## **三个核心概念**
### **1. Promise 是什么？**
可以把 `Promise<T>` 理解为一个代表未来结果的对象：结果可能成功，也可能失败。
```typeScript
function getUser(): Promise<string> {
  return new Promise((resolve) => {
    setTimeout(() => resolve("Alice"), 500);
  });
}
```
这里的 `getUser()` 不会立即返回字符串，而是返回 `Promise<string>`。
### **2. async/await 是什么？**
```typeScript
async function main(): Promise<void> {
  const user = await getUser();
  console.log(user);
}
```

- `async`：让函数返回 Promise；即使函数返回普通值，也会被包装为已兑现的 Promise。
- `await`：等待 Promise 的结果，让后续代码以更接近同步代码的形式编写。
- `await`：不会阻塞整个 Node.js 线程。等待期间，事件循环仍然可以处理其他任务。
Java 对照：它不等同于 `Future.get()` 在调用线程上阻塞等待的行为。
### **3. Promise.all() 为什么重要？**
假设 BFF 需要同时调用 Java 的用户服务和订单服务：
```typeScript
const [user, orders] = await Promise.all([
  getUser(),
  getOrders()
]);
```

两个请求可以并行进行，不必等待用户服务返回后才开始调用订单服务。
注意：`Promise.all()` 中任意一个 Promise 拒绝，整体就会拒绝；它也不会自动取消其他已经发出的请求。
