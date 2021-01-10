## vm.$emit(eventName,[...args])
#### 一般用于子组件调用父组件的方法，使用eventName作为事件名，附加的参数都会传给监听器回调。

- 示例

只配合一个事件名使用` $emit`：

    Vue.component('welcome-button', {                                            jS
      template: `
        <button v-on:click="$emit('welcome')">
          Click me to be welcomed
        </button>
      `
    })


------------
    <div id="emit-example-simple">                                               HTML
      <welcome-button v-on:welcome="sayHi"></welcome-button>
    </div>
------------
    new Vue({                                                                    JS
      el: '#emit-example-simple',
      methods: {
        sayHi: function () {
          alert('Hi!')
        }
      }
    })

------------

配合额外的参数使用 `$emit`，`$emit`中的额外参数传入到父组件函数中，作为父组件函数的参数，或者我们可以通过`$event`访问到被抛出的这个值：

        Vue.component('magic-eight-ball', {                                       JS
          data: function () {
            return {
              possibleAdvice: ['Yes', 'No', 'Maybe']
            }
          },
          methods: {
            giveAdvice: function () {
              var randomAdviceIndex = Math.floor(Math.random() * this.possibleAdvice.length)
              this.$emit('give-advice', this.possibleAdvice[randomAdviceIndex])
            }
          },
          template: `
            <button v-on:click="giveAdvice">
              Click me for advice
            </button>
          `
        })
		

------------

    <div id="emit-example-argument">                                            HTML
      <magic-eight-ball v-on:give-advice="showAdvice"></magic-eight-ball>
    </div>
	

------------

    new Vue({                                                                    JS
      el: '#emit-example-argument',
      methods: {
        showAdvice: function (advice) {
          alert(advice)
        }
      }
    })