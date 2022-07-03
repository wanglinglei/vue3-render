# Vue3 渲染器
## 什么渲染器？
简单的说，渲染器就是执行渲染任务的，在浏览器平台上把虚拟的dom节点渲染为真实的dom元素
```
const vnode={
  tag:'h1',
  children:'h1标签'
}

function render(vnode){
  const ele=document.createElement(vnode.tag);
  ele.textContent=vnode.children;
  document.body.appendChild(ele);
}
```
上面的代码render函数就是一个简单的渲染器，同过render函数我们把vnode这个虚拟节点插入到了页面中，完成了虚拟dom到真实dom的转变。