## 第一章 React入门

### 1.1 react基本知识

#### 1.1.1 虚拟DOM、diff算法

#### 1.1.2 jsx语法简单规则

1. 定义虚拟DOM时，不要写引号
2. 标签中混入JS表达式时要用{}
3. 样式的类名指定不要用class，要用className
4. 内联样式，要用style={{key:value}}的格式
5. 虚拟DOM必须只有一个根标签
6. 标签必须闭合
7. 标签首字母
   1. 若小写字母开头，则将该标签转为HTML中的同名元素。若无，则则报错
   2. 若大写字母开头，react渲染对应的组件，若无，则报错



**样例**

```html
<html>
    <head>
        <title></title>
    </head>
    <body>
        <div id="text"></div>
    </body>
</html>
```



```jsx
const VDOM = (
	<div>
        <h1>前端框架列表</h1>
        <ul>
            <li>Angular</li>
            <li>React</li>
            <li>Vue</li>
        </ul>
    </div>
)
```



```react
ReactDOM.render(VDOM, document.getElementById('text'))
```



#### 1.1.3 函数式组件

```react
function Demo() {
	return <h2>我是函数定义的组件</h2>
}
ReactDom.render(<Demo/>, document.getElementById('test'))
```

执行ReactDom.render(<Demo/>, document.getElementById('test'))后发生了什么？

1. React解析组件标签，找到Dome组件。
2. 发现组件是使用函数定义的，随后调用该函数，将返回的虚拟DOM转为真实DOM，随后呈现在页面中。



#### 1.1.4 类式组件

```react
class MyComponent extents React.Component {
    //render放在MyComponent的原型对象上，供实例使用
    //render中的this是谁？ MyComponent的实例对象 <=> myComponent组件实例对象
	render() {
		return <h2>我是类定义的组件（复杂组件）</h2>
	}
}

ReactDom.render(<MyComponent/>, document.getElementById('test'))
```

执行ReactDom.render(<Demo/>, document.getElementById('test'))后发生了什么？

1. React解析组件标签，找到Dome组件。
2. 发现组件是使用类定义的，随后new出该类的实例，并通过该实例调用原型 上的render方法
3. 将render返回的虚拟DOM转为真实DOM，随后呈现在页面中。



### 2.1 组件（实例）的三大核心属性

#### 2.1.1 state

通过是否有无state属性值，判断简单组件和复杂组件

```react
class Weather extends React.Component {
    
    // constructor调用几次？ ———— 1次
	constructor(props) {
		super(props);
		this.state = {
			isHot: true
		}
        // 解决changeWeather中的this指向问题
        this.changeWeather = this.changeWeather.bind(this);
	}
    
	// render调用几次？ ———— 1+n次 1是初始化那次，n是状态更新的次数
	render() {
		return {
			const {isHot} = this.state;
			<h1 onClick={this.changeWeather}>今天天气很{isHot ? '炎热' : '凉爽'}</h1>
		}
	}		
	
	changeWeather() {
        //changeWeather放在原型对象上，供实例使用
        //由于changeWeather是作为onClick的回调，所以不是通过实例调用的，而实直接调用的。
        //类中的方法默认开启了局部的严格模式，所以changeWeather中的this指向undefined
        console.log(this);
        
        // 获取原来的isHot值
        const isHot = this.state.isHot;
        // state不可直接更改（this.state.isHot = !isHot 错误写法）
        // state必须通过setState更改
        this.setState({isHot:!isHot})
       
    }
}

ReactDom.render(<Weather/>, document.getElementById("test"));
```



```react
// 简化组件创建
class Weather extends React.Component {
	//初始化状态
    state = {isHot: true}
    
	render() {
		return {
			const {isHot} = this.state;
			<h1 onClick={this.changeWeather}>今天天气很{isHot ? '炎热' : '凉爽'}</h1>
		}
	}		
	
	//自定义方法————赋值语句+箭头函数
	changeWeather = () => {
        const isHot = this.state.isHot;
        this.setState({isHot:!isHot})
    }
}

ReactDom.render(<Weather/>, document.getElementById("test"));
```



#### 2.1.2 props

```
class Person extends React.Component{
	
    //对标签属性进行类型、必要性的限制
    static propTypes = {
        name:PropType.string.isRequired,
        sex: PropType.string,
        age: PropType.number
    }

    //指定默认标签属性值
    static defaultProps = {
        sex: '男'，
        age:18
    }

	render() {
		const {name, age, sex} = this.props;
		return {
			<ul>
				<li>{name}</li>
				<li>{age}</li>
				<li>{sex}</li>
			</ul>
		}
	}
}

ReactDom.render(<Person name="tom" age={18} sex="男"/>, document.getElementById('test'));

const p = {name:'老刘', age:18, sex:'女'}；
ReactDom.render(<Person {...p} />, document.getElementById('test2'))

```

