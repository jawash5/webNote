# Promise笔记

### 异步编程

- 文件操作

  ```
  require('fs').readFile('.index,html', (err,data)=>{});
  ```

- 数据库操作

- AJAX

  ```
  $.get('/server', (data)=>{});
  ```

- 定时器

  ```
  setTimeout(()=>{}, 2000);
  ```

  
  
### Promise 的状态

实例对象中的属性 PromiseState

- pending  未决定的
- resolved / fullfilled   成功
- rejected  失败



### Promise 对象的值

实例对象中的另一个属性 PrimiseResult

保存着异步任务 成功、失败 的结果



### 手写Promise

```javascript
class Promise {
	//构造方法
	constructor(executor) {
		//添加属性
		this.PromiseState = 'pending';
		this.PromiseResult = null;
		//声明属性
		this.callbacks = [];	
		//保存实例对象的 this 的值
		const self = this;
		
		//resolve 函数
		function resolve(data) {
			// 判断状态
			if(self.PromiseState !== 'pending') return;
			// 1. 修改对象的状态（PromiseState）
			self.PromiseState = 'fulfilled';
			// 2. 设置对象的结果值 （PromiseResult）
			self.PromiseResult = data;
			//调用成功的回调函数
			setTimeout( () => {
				self.callbacks.forEach(item => {
					item.onResolved(data);
				});
			});	
		}
		
		//reject 函数
		function reject(data) {
			// 判断状态
			if(self.PromiseState !== 'pending') return;
			// 1. 修改对象的状态（PromiseState）
			self.PromiseState = 'rejected';
			// 2. 设置对象的结果值 （PromiseResult）
			self.PromiseResult = data;
			//调用失败的回调函数
			setTimeout(() => {
				self.callbacks.forEach(item => {
					item.onRejected(data);
				});
			});
		}
		
		try {
			//同步调用【执行器函数】
			executor(resolve, reject);
		} catch(e) {
			//修改 promise 对象状态为【失败】
			reject(e);
		}
	}
	
	//then 方法封装
	then(onResolved, onRejected) {
		const self = this;
		//异常穿透
		if(typeof onResolved !== 'function') {
			onResolved = value => {
				throw value;
			}
		}
		//判断回调函数参数、异常穿透
		if(typeof onRejected !== 'function') {
			onRejected = reason => {
				throw reason;
			}
		}
		return new Promise((resolve, reject) => {
	
			function callback(type) {
				try{
					//获取回调函数的执行结果
					let result = type(self.PromiseResult);
					//判断
					if(result instanceof Promise) {
						//如果是Promise类型的对象
						result.then( v => {
							resolve(v);
						}, r => {
							reject(r);
						});
					} else {
						//结果的对象状态为【成功】
						 resolve(result);
					}
				} catch(e) {
					reject(e);
				}
			}
			
			// 调用回调函数
			if(this.PromiseState === 'fulfilled') {
				//异步执行
				setTimeout( () => {
					callback(onResolved);
				})
			}
			if(this.PromiseState === 'rejected') {
				//异步执行
				setTimeout( () => {
					callback(onRejected);	
				})
			}
			//判断pending的状态
			if(this.PromiseState === 'pending') {
				//保存回调函数
				this.callbacks.push({
					onResolved: function() {
						callback(onResolved);
					},
					onRejected: function() {
						callback(onRejected);
					}
				});
			}
		})
	}
	
	//catch 方法 
	catch(onRejected) {
		return this.then(undefined, onRejected);
	}
	
	//添加resolve方法
	static resolve(value) {
		//返回promise对象
		return new Promise((resolve, reject) => {
			if(value instanceof Promise) {
				value.then( v => {
					resolve(v);
				}, r => {
					reject(r);
				})
			} else {
				//设置状态为成功
				resolve(value);
			}
		})
	}
	
	//添加reject方法
	static reject(reason) {
		//返回promise对象
		return new Promise((resolve, reject) => {
			reject(reason);
		})
	}
	
	//添加all方法
	static all(promises) {
		//返回结果为promise对象
		return new promise((resolve, reject) => {
			//声明遍历
			let count = 0;
			let arr = []; 
			//遍历
			for(let i=0; i<promises.length; i++) {
				promises[i].then( v => {
					//得知对象的状态是成功
					count++;
					//将当前 promise 对象成功的结果存入到数组中
					arr[i] = v;
					if(count === promises.length) {
						resolve(arr);
					}
				}, r => {
					reject(r);
				})
			}
		})
	}
	
	//添加race方法
	static race(promises) {
		return new Promise((resolve, reject) => {
			for(let i=0; i<promises.length; i++) {
				promises[i].then(v => {
					//修改返回对象的状态为 【成功】
					resolve(v);
				}, r => {
					//修改返回对象的状态为 【失败】
					reject(r);
				})
			}
		})
	}
}

```

