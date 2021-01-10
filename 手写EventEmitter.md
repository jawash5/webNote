#### 手写EventEmitter

最近发现好多大厂面试题里都有手写发布订阅模式 EventEmitter，原因是 vue 和 react 的非父子组件的通信就是靠他实现的，下面是一个简易版的 EventEmitter。

代码如下：

```javascript
// 发布订阅模式
class EventEmitter {
  constructor() {
    // 事件对象，存放订阅的名字和事件  如:  { click: [ handle1, handle2 ]  }
    this.events = {}
  }
  // 订阅事件的方法
  on(eventName, callback) {
    if (!this.events[eventName]) {
      // 一个名字可以订阅多个事件函数
      this.events[eventName] = [callback]
    } else {
      // 存在则push到指定数组的尾部保存
      this.events[eventName].push(callback)
    }
  }
  // 触发事件的方法
  emit(eventName, ...rest) {
    // 遍历执行所有订阅的事件
    this.events[eventName] &&  //判断事件是否存在
      this.events[eventName].forEach(f => f.apply(this, rest))
  }
  // 移除订阅事件
  remove(eventName, callback) {
    if (this.events[eventName]) {
      this.events[eventName] = this.events[eventName].filter(f => f != callback)
    }
  }
  // 只执行一次订阅的事件，然后移除
  once(eventName, callback) {
    // 绑定的时fn, 执行的时候会触发fn函数
    const fn = () => {
      callback() // fn函数中调用原有的callback
      this.remove(eventName, fn) // 删除fn, 再次执行的时候之后执行一次
    }
    this.on(eventName, fn)
  }
}
```



```
class EventEmitter {
	constructor() {
		this.events = {}; //event是个对象
	}
	
	on(eventName, callback) {
		if(!this.events[eventName]) {
			this.events[eventName] = [callback];
		} else {
			events[eventName].push(callback);
		}
	}
	
	emit(eventName, ...rest) {
		if(this.events[eventName]) {
			this.events[eventName].foreach(f => f.apply(this, rest))
		}
	}
	
	off(eventName, callback) {
		if(this.events[eventName]) {
			this.events[eventName] = this.events[eventName].filter(f => f != callback)
		}
	}
	
	once(eventName, callback) {
		const a = () => {
			callback();
			this.off(eventName, callback);
		}
		this.on(eventName, callback);
	}
}
```



