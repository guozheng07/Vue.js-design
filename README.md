《Vue.js 设计与实现》导读
# 4.响应系统的作用与实现
## 响应系统
概念
- 副作用函数 effect 是什么？
- 什么是响应式数据？
- 什么叫可调度性？-> trigger 动作触发副作用函数重新执行时，有能力决定副作用函数执行的时机、次数以及方式  

设计一个完整的响应式系统需要解决的问题
- 在副作用函数与被操作的目标字段之间建立明确的关系 -> weakMap + Map
- 避免分支切换时可能会产生遗留的副作用函数的问题 -> Set
- 副作用函数发生嵌套时，响应式数据在外层副作用函数中读取，收集到的副作用函数也都是内层的副作用函数 -> effect 栈
- 无限递归循环导致的栈溢出 -> trigger 触发执行的副作用函数与当前正在执行的副作用函数相同，则不触发执行。
- 怎么实现调度能力？-> 为 effect 添加选项参数，通过 scheduler 选项指定调度器，并通过调度器实现任务去重。

## 实现 computed
概念
- 懒执行的 effect 是什么意思？
- 实质？ -> 一个懒加载的副作用函数（通过手动方式使其执行）  

问题
- 怎么利用 lazy effect，实现懒计算？
- 怎么实现值缓存？
- 所依赖的响应式数据变化时，怎么调用副作用函数重新进行计算？-> 通过 scheduler 将 dirty 标记设置为 true（脏），下次读取计算属性的值时重新计算。
- 在另一个副作用函数中读取计算属性（不是在该副作用函数中读取计算属性的值）且计算属性所依赖的响应式数据变化时，没有触发该副作用函数的重新执行，为什么？怎么解决？
  - 原因：本质上一个典型的 effect 嵌套。一个计算属性内部拥有自己的 effect，并且它是懒执行的，只有当真正读取计算属性的值时才会执行。对于计算属性的 getter 函数来说，它里面访问的响应式数据只会把 computed 内部的 effect 收集为依赖。而当把计算属性用于另外一个 effect时，就会发生 effect 嵌套，外层的 effect 不会被内层 effect 中的响应式数据收集。
  - 解决：当读取计算属性时，手动调用 track 函数进行追踪；当计算属性依赖的响应式数据发生变化时，可以手动调用 trigger 函数触发响应。  

## 实现 watch
概念
- 在多进程或多线程中的竞态问题是什么？
- 实现原理？ -> 本质上利用了副作用函数重新执行时的可调度行。  

设计 watch 函数
- 将硬编码式的的读取操作（例如只能读取对象 a 的属性 b）封装成通用的读取操作 -> traverse 函数进行递归读取
- 怎么把 watch 函数的第一个参数从一个响应式数据变成一个 getter 函数（从观测一个响应式数据深入到接收一个自定义函数，使 watch 更强大）？->  定义 getter 来区分两种情况
- 回调函数中怎么拿到新值和旧值？-> 使用 lazy 选项 创建一个懒加载的 effect 函数的
- 怎么选择 watch 回调的执行时机 ？-> 选项参数 + 把 scheduler 调度器封装成一个通用函数
  - 立即执行 -> immediate 选项参数
  - 其他执行时机 -> flush 选项参数（本质上是利用了调度器和异步的微任务队列）
- 怎么解决过期的副作用？-> watch 的第三个参数 onInvalidate，使用 onOnvalidate 函数注册一个回调，这个回调函数在当前副作用函数过期时执行（每当 watch 的回调函数执行之前，会优先执行用户通过 onInvalidate 注册的过期回调。这样，用户就有机会在过期回调中将上一次的副作用标记为“过期”，从而解决竞态问题）

# 5.非原始值的响应式方案
## 概念
- 什么是代理、Proxy、Reflect？基本操作有哪些？非基本操作有哪些？
- Reflect 函数的作用？（见86～88页）
- 怎么区分一个对象是普通对象还是函数？怎么区分一个对象是普通对象还是异质对象？
- 什么是浅响应？-> 只是对象的第一层属性是响应的。  

## 代理 Object
- 对一个普通对象读取操作的拦截
- 访问属性的拦截 -> Proxy 构造函数中设置 get 函数来 track，
- in 操作符的拦截 -> Proxy 构造函数中设置 has 函数来 track
- for...in 循环的拦截 -> Proxy 构造函数中设置 ownKeys 函数来 track
  - 新增属性，会增加 for...in 循环次数 -> Proxy 构造函数中设置 set 函数来 track && 需要触发与 ITERATE_KEY 相关联的副作用函数重新执行 -> 完善 trigger 函数（新增 type 参数）
  - 修改属性，for...in 循环次数不变 -> Proxy 构造函数中设置 set 函数来 track && 不需要触发与 ITERATE_KEY 相关联的副作用函数重新执行 -> 完善 trigger 函数（新增 type 参数）
  - 删除属性，会减少 for...in 循环次数 -> Proxy 构造函数中设置 set 函数来 track && 需要触发与 ITERATE_KEY 相关联的副作用函数重新执行 -> 完善 trigger 函数（新增 type 参数）
- 合理地触发响应
  - 设置的值没有发生变化时，不需要触发响应（另一种说法：“设置的值发生变化时，并且都不是 NaN时，需要触发响应”）-> 完善 Proxy 构造函数中的 set 函数
  - 将 Proxy 构造函数进行封装 -> 得到组合式 API 中熟悉的 reactive 方法
  - 屏蔽由原型引起的更新 -> 确定 receiver 是不是 target 的代理对象（只有 receiver 是 target 的代理对象才更新） -> 完善 Proxy 构造函数中的get 函数（raw 属性）
- 浅响应和深响应
  - 怎么实现深响应？-> Reflect.get 返回的结果是对象，则递归地调用 reactive 函数将其包装成响应式对象，再返回 -> 完善 Proxy 构造函数中的 get 函数
  - 怎么兼顾深响应和浅响应？-> reactive 函数摇身一变成 createReactive 函数，并完善 Proxy 构造函数中的 get 函数（添加 isShallow 属性），实现深响应（reactive）和浅响应（shallowReactive）时分别调用 createReactive 函数，赋予不同的 isShallow 参数
- 只读和浅只读
  - 怎么实现只读？-> 为 createReactive 函数添加 isReadonly 属性 -> 完善 Proxy 构造函数中的 set 函数（不允许修改）和 deleteProperty（不允许删除）

# 6.原始值的响应式方案
## 原始值
概念
- 有哪些？
- 有什么特点？

## 引入 ref
概念
- 什么是响应丢失问题？-> 把一个普通对象暴露到模版中使用

实现 ref
- JS 中的 Proxy 无法提供对原始值的代理，怎么解决？-> 使用非原始值去“包裹”原始值
- 区分一个 ref 是原始值的包裹对象，还是一个非原始值的响应式数据？-> 通过 Object.defineProperty 为包裹对象 wrapper 定义 __v_isRef 属性
- 怎么解决响应丢失问题？-> 将对象改造成访问器对象，在副作用函数内读取 newObj.foo 时，等价于间接读取了 obj.foo -> 抽象结构，得到 toRef 函数
- 响应式数据 obj 的键非常多 -> 封装 toRefs 函数
- 通过 toRef 函数创建的 ref 是只读的 -> 为 toRef 函数返回对象的 value 属性设置 setter
- ref 的作用：实现原始值的响应式方案、解决响应丢失问题

## 自动脱 ref
- toRefs 函数有什么问题？-> 必须通过 value 属性访问值
- 自动脱 ref 的含义？-> 如果读取的属性是一个 ref，则直接将该 ref 对应的 value 属性值返回
- 怎么实现自动脱 ref？-> proxyRefs 函数（使用 Proxy 为 newObj 创建一个代理对象，拦截 set 和 get 操作）
- 为什么我们可以在模版直接访问一个 ref 的值，而无需通过 value 属性来访问？-> setup 返回的对象会传递给 proxyRefs
- 自动脱 ref 的意义？-> 用户在模版中使用响应式数据时，将不再关心哪些是 ref，哪些不是 ref。

# 7.渲染器的设计
## 渲染器
概念
- 挂载、容器
- 渲染器与渲染的区分  

功能
- 作用
- 响应式系统与渲染器的关系

## 自定义渲染器的实现
概念
- 渲染函数：render
- 更新函数：patch
- 挂载函数：mountElement  

实现
- 怎么渲染一个普通标签元素？-> 将 render、patch、mountElement 函数串联起来
- 怎么设计一个不依赖于浏览器平台的通用渲染器？-> 将浏览器特有的 API 抽离 -> 将操作 DOM 的 API 作为配置项（新增、修改、删除等），该配置项作为 createRenderer  函数的参数，这样在 mountElement  函数中就可以通过配置项来获取取得 DOM 的 API（不需要在本身函数中写死）。  

本质
- 自定义渲染器只是通过“抽象”的手段，让核心代码不再依赖平台特有的 API，再通过支持个性化配置的能力来实现跨平台。

# 8.挂载与更新
## 挂载
概念
- 完整的虚拟 DOM 有什么内容？
  - type：元素（div、span 等）
  - props：元素的属性（通用-id、class；特定-表单的 action 等）
  - children：元素的子节点（元素的子节点除了文本节点，还可以包含其他元素节点，因此 vnode.children 是一个数组）
- HTML Attributes 与 DOM Properties 的定义、联系和区别？-> HTML Attributes 的作用是设置与之对应的 DOM Properties 的初始值。
- 以 disabled 属性为例，说明为什么使用 setAttribute 函数还是直接设置元素的 DOM Properties 均存在缺陷？-> 虚拟 DOM 的属性值为空字符串，带来的问题（185页）怎么解决？-> 正确地设置元素属性
- 为什么对 class 属性做特殊处理？-> Vue.js 对 class 属性做了增强，可以设置字符串、对象和数组，所以必须在设置元素的 class 之前将值归一化为统一的字符串形式，再把该字符串作为元素的 class 值去设置。
- 在浏览器中为一个元素设置 class 有三种方式：使用 setAttribute、el.className（性能最优，所以用这种方式设置 class）、el.classList。  

实现
- vnode.children 是一个数组，怎么改写 渲染器中的 mountElement/挂载 函数？-> 增加 Array.isArray(vode.children)的判断条件
- 怎么将元素的属性渲染到对应的元素上？-> 增加 vnode.props 的遍历 + 调用 setAttribute 设置/通过 DOM 对象直接设置
- 怎么正确地设置元素属性？-> 优先设置 DOM Properties；同时对布尔类型的 DOM Properties 做值的矫正（当要设置的值为空字符串时设置成 true）；只读的 DOM Properties（input 标签的 form 属性） 通过 setAttribute 函数设置；把属性的设置变成与平台无关。
- 怎么对 class 属性做特殊处理？-> 在设置元素的 class 之前将值归一化为统一的字符串形式（封装 normalizeClass 函数对值进行序列化）；使用 el.className 设置 class 属性。
- Vue.js 对 style 属性也做了增强，所以对 style 属性也做类似于 class 的处理。

## 卸载
概念
- 《自定义渲染器》小节中，vode 为 null 且 container._vnode 属性存在时，直接通过 innerHTML 清空容器为什么不严谨？
  - 容器的内容可能由某个或多个组件渲染，卸载时应调用这些组件的 beforeUnmount、unMounted 等生命周期函数。
  - 即使内容不是由组件渲染的，有的元素存在自定义指令，应该在卸载时正确执行对应的指令钩子函数。
  - 使用 innerHTML 清空容器元素内容，不会移除绑定在 DOM 元素上的事件处理函数。
- 将卸载操作封装到 unmount 中，有什么好处？
  - 在 unmount 函数内，有机会调用绑定在 DOM 元素上的指令钩子函数
  - 在 unmount 函数执行时，有机会检测虚拟节点 vnode 的类型，若虚拟节点是描述的是组件，有机会调用组件相关的生命周期函数。  

实现
- 正确的卸载方式是什么？-> 根据 vnode 对象获取与其相关联的真实 DOM 元素，然后使用原生 DOM 操作方法将 DOM 元素删除 -> 在 vnode 与真实 DOM 元素间建立联系，修改 mountElement 函数和 render 函数，封装 unmount 函数。  

## 更新
概念
- 一个 vnode 可以描述的类型：描述普通标签、描述组件、描述 Fragment 等。
- 发生事件冒泡时，父元素绑定事件函数发生在事件冒泡之前。这种现象导致了初始化时本来没有绑定事件的父元素绑定了事件并执行（详情见202页）。
- vnode.children 有三种情况：没有子节点、具有文本子节点、其他情况。所以更新子节点共有 3 * 3 = 9 种情况。
- 注释节点、文本节点、Fragment 不同于普通标签节点，需要人为创造一些唯一标识（通过 Symbol），作为它们的 vnode.type 属性值
  - 文本节点：type 为 Text（const Text = Symbol()）
  - 注释节点：type 为 Comment（const Comment = Symbol()）
  - Fragment：type 为 Fragment（const Fragment = Symbol()）
- Fragment（片段）是 Vue.js 3 新增的一个 vnode 类型（通过 Fragment，Vue.js 3实现了支持多根节点模版，这是 Vue.js 2不支持的）（参考212页）。
- 对于 Fragment 类型的 vnode，它的 children 存储的内容就是模版中所有根节点。  

普通标签和事件更新的实现
- 区分 vnode 的类型？-> 新旧 vnode 描述的内容相同时，需要打补丁；新旧 vnode 描述的内容不同时，需要先卸载旧的元素，再挂载新的元素，需要调整 patch 函数（对比 n1 和 n2 的类型） 
- 新旧 vnode 描述的内容相同，如何进一步确定它们的类型是否相同？-> 调整 patch 函数（根据 n2 的类型分别进行处理）
- 怎么处理事件 -> 改造 patchProps
  - 如何在虚拟节点中描述事件？-> 约定在 vnode.props 中，以字符串 on 开头的属性都视为事件
  - 如何把事件添加到 DOM 元素上？-> 在 patchProps 中调用 addEventListener 函数来绑定事件
  - 如何更新事件？-> 方式一：移除之前添加的事件处理函数，然后将新的事件处理函数绑定到 DOM 元素上；方式二：绑定事件时，绑定一个伪造的事件处理函数 invoker，把真正的事件处理函数设置为 invoker.value 属性的值，在更新事件时，不再需要调用 removeEventListener 函数来移除上一次绑定的事件，只需要更新 invoker.value 属性的值即可（性能更优，避免一次 removeEventListener 函数的调用）
  - 同一个元素同时绑定了多种事件时，怎么解决事件覆盖的现象（事件处理函数都缓存在 el._vei 中，而同一时刻只能缓存一个事件处理函数）？-> 重新设计  el._vei 的数据结构（改为对象）
  - 同一类型的事件绑定了多个事件处理函数应该怎么解决（例如为 click 事件同时绑定了回调 fn1 和 fn2）？-> 增加 invoker.value 是数组的处理逻辑
- 怎么解决“父元素绑定事件函数发生在事件冒泡之前”导致的问题？-> 因为事件触发的时间要早于事件处理函数被绑定的事件，利用这个特点，可以通过“屏蔽所有绑定时间晚于事件触发时间的事件处理函数的执行”来解决 -> 增加 invoker.attached 属性  

元素子节点更新的实现
- 怎么更新子节点？-> 根据新旧 vnode.children 的类型分情况处理（理论有9种情况，实际上有些情况可以合并处理或不需要处理） -> patchElement 函数和 patchChildren 函数  

文本节点和注释节点的渲染
- 除了描述普通标签， vnode 怎么描述文本节点和注释节点？-> 完善 patch 函数
- patch 函数依赖平台特有的 API，保证跨平台将操作 DOM 的 API 封装到渲染器的选项中 -> 增加调用 createRenderer 函数创建 renderer 时的选项配置  

Fragment 的渲染
- 从本质上来说，渲染 Fragment 与渲染普通元素的区别是：Fragment 本身不渲染任何内容，只需要处理它的子节点即可。-> 完善 patch 函数
- 卸载也需要支持 Fragment 类型虚拟节点的卸载 -> 完善 unmount 函数
