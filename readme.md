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

##### 实现一个基本的渲染器
```
  const methods = {
    createElement(tag) {
      return document.createElement(tag);
    },
    setElementText(ele, text) {
      ele.textContent = text;
    },
    insert(ele, container) {
      container.appendChild(ele);
    },
  };
  // 定义一个创建渲染器的方法
  function createRenderer(methods) {
    const { createElement, setElementText, insert } = methods;
    // 接收三个参数 n1 老节点 n2 新节点 container
    function patch(n1, n2, container) {
      // 新老节点一样 不需要渲染
      if (n1 === n2) {
        return;
      }
      // n1 不存在初次渲染
      if (!n1) {
        mountElement(n2, container);
      }
    }
    // 节点初始化挂载方法
    function mountElement(vnode, container) {
      const ele = createElement(vnode.tag);
      // 如果是string类型 则渲染文本 如果是array类型 渲染子节点
      if (typeof vnode.children === "string") {
        setElementText(ele, vnode.children);
      } else if (Array.isArray(vnode.children)) {
        vnode.children.forEach((item) => {
          patch(null, item, ele);
        });
      }
      insert(ele, container);
    }
    // render 函数接受两个参数 vnode 虚拟dom节点 container 节点挂载的容器
    function render(vnode, container) {
      if (vnode) {
        // 执行挂载方法
        patch(container._vnode, vnode, container);
      } else {
        // 新节点不存在 说明节点被删除
        if (container._vnode) {
          container.innerHtml = "";
        }
      }
      container._vnode = vnode;
    }
    return { render };
  }
```

## 渲染器实现流程简述
####普通HTML节点渲染
1. 根据虚拟dom节点类型创建对应dom
   ```
   const ele = document.createElement(tag);
   ```
2. 绑定对应属性
    * 如果节点的children属性是string，则为文本,直接设置文本属性
      ``` 
      ele.textContent=text
      ```
    * 如果节点children属性是array类型，则为子节点,递归patch函数
      ```
      vnode.children.forEach((item) => {
        patch(null, item, ele);
      });
      ```
    * 如果节点的props属性有值，需要绑定dom属性,这里需要判断属性值是否属于DOM properties,属于的话通过key值设置，不属于通过setAttribute设置
      ```
      function shouldSetAsProps(ele, key, value) {
        // 特殊处理 如果是input标签 form 属性是只读属性 不能通过ele[key]来设置
        if (key === "form" && ele.tagName === "INPUT") {
          return false;
        }
        // 用in操作符判断key是否存在于对应的DOM Properties
        return key in ele;
      }
      ``` 
      ```
      patchProps(ele, props, value) {
        if (key === "class") {
          ele.className = value;
        }
        if (shouldSetAsProps(ele, key)) {
          const type = typeof ele[key];
          //这里主要处理 <input disable/> 这种情况
          if (type === "boolean" && value === "") {
            ele[key] = true;
          } else {
            ele[key] = value;
          }
        } else {
          ele.setAttribute(key, props[key]);
        }
      },
      ```
    *  
3. 绑定事件
    * 基本实现 通过正则匹配 **on** 关键字来获取事件名称
      ```
      patchProps(ele, key, value) {
        if (/^on/.test(key)) {
          // 获取事件名称
          const event = key.slice(2).toLowerCase();
          ele.addEventListener(event, value);
        }
        ...省略其他内容
      },
      ```
    * 绑定事件的更新
      ```
      if (/^on/.test(key)) {
        const invoker = ele._eventInvoker;
        // 获取事件名称
        const event = key.slice(2).toLowerCase();
        if (nextValue) {
          // 实际绑定的是invoker函数 执行时执行的是invoker.value 因此当我们需要更新时只需要更新value 值
          if (!invoker) {
            // 没有invoker 生成并挂载
            invoker = ele._eventInvoker = (e) => {
              invoker.value(e);
            };
            invoker.value = nextValue;
            ele.addEventListener(event, invoker);
          } else {
            invoker.value = nextValue;
          }
        } else {
          // 不存在新的函数  但存在invoker 移除绑定
          if (invoker) {
            ele.addEventListener(event, invoker);
          }
        }
      }
      ``` 
    * 绑定了多个事件类型以及一个类型绑定了多个事件
      ```
      if (/^on/.test(key)) {
        // 定义一个键值对 来实现多个事件类型的绑定 避免事件覆盖
        const invokers = ele._eventInvoker || (ele._eventInvoker = {});
        let invoker = invokers[key];
        // 获取事件名称
        const event = key.slice(2).toLowerCase();
        if (nextValue) {
          // 实际绑定的是invoker函数 执行时执行的是invoker.value 因此当我们需要更新时只需要更新value 的值
          if (!invoker) {
            // 没有invoker 生成并挂载
            invoker = ele._eventInvoker[key] = (e) => {
              // 如果是数组类型 说明同一个事件绑定了多个函数
              if (Array.isArray(invoker.value)) {
                invoker.value.forEach((fn) => fn());
              } else {
                invoker.value(e);
              }
            };
            invoker.value = nextValue;
            ele.addEventListener(event, invoker);
          } else {
            invoker.value = nextValue;
          }
        } else {
          // 不存在新的函数  但存在invoker 移除绑定
          if (invoker) {
            ele.addEventListener(event, invoker);
          }
        }
      }
      ```  
4.   
