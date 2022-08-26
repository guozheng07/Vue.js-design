# 减少 DOM 操作的性能开销
“更新子节点”小节中，对于新旧 vnode 的 children 都是一组子节点，做了傻瓜式的处理，这里基于减少 DOM 操作的前提下进行改进：
```
function patchChildren(n1, n2, container) {
	if (typeof n2.children === 'string') {
    if (Array.isArray(n1.children)) {
    	n1.children.forEach((c) => c.unmount(c))
    }
    setElementText(container, n2.children)
  } else if (Array.isArray(n2.children)) {
  	if (Array.isArray(n1.children)) {
    	// 重新实现两组子节点的更新方式
      // 新旧 children
      const oldChildren = n1.children
      const newChildren = n2.children
      const oldLen = oldChildren.length
      const newLen = newChildren.length
      // 两组子节点的公共长度，即两者中较短的那一组子节点的长度
      const commonLength = Math.min(oldLen, newLen)
      // 遍历 commonLength 次
      for (let i = 0;i < commonLength; i++) {
      	// 调用 patch 函数逐个更新子节点
        patch(oldChildren[i], newChildren[i])
      }
      // 如果 newLen > oldLen，说明有新节点需要挂载
      if (newLen > oldLen) {
      	for (let i = commonLength;i < newLen; i++) {
        	patch(null, newChildren[i], container)
      	}
      } else if (oldLen > newLen) {
      	// 如果 oldLen > newLen，说明有旧节点需要卸载
        for (let i = commonLength;i < oldLen; i++) {
        	unmount(oldChildren[i])
      	}
      }
    } else {
      setElementText(container, '')
      n2.children.forEach(c => patch(null, c, comtainer))
    }
  } else { 
    if (Array.isArray(n1.children)) {
      n1.children.forEach(c => unmount(c))
    } else if (typeof n1.children === 'string') {
    	setElementText(container, '')
    }
  }
}
```
# DOM 复用与 key 的作用
引入 key 后，改写 patchChildren 函数：
```
function patchChildren(n1, n2, container) {
	if (typeof n2.children === 'string') {
    if (Array.isArray(n1.children)) {
    	n1.children.forEach((c) => c.unmount(c))
    }
    setElementText(container, n2.children)
  } else if (Array.isArray(n2.children)) {
  	if (Array.isArray(n1.children)) {
      const oldChildren = n1.children
      const newChildren = n2.children
      
      // 遍历新的 children
      for (let i = 0; i < newChildren.length; i++) {
      	const newVNode = newChildren[i]
        // 遍历旧的 children
        for (let j = 0; j < oldChildren.length; j++) {
        	const oldVNode = oldChildren[j]
          // 如果找到了具有相同 key 值的两个节点，说明可以复用，但仍需要调用 patch 函数进行更新
          if (newVNode.key === oldVNode.key) {
          	patch(oldVNode, newVNode, container)
            break
          }
        }
      }
    } else {
      setElementText(container, '')
      n2.children.forEach(c => patch(null, c, comtainer))
    }
  } else { 
    if (Array.isArray(n1.children)) {
      n1.children.forEach(c => unmount(c))
    } else if (typeof n1.children === 'string') {
    	setElementText(container, '')
    }
  }
}
```
# 移动元素
找到需要移动的元素：用 lastIndex 存储整个寻找过程中遇到的最大索引值。
移动元素：通过旧子节点的 vnode.el 属性取得它对应的真实 DOM 节点。
```
function patchChildren(n1, n2, container) {
	if (typeof n2.children === 'string') {
    if (Array.isArray(n1.children)) {
    	n1.children.forEach((c) => c.unmount(c))
    }
    setElementText(container, n2.children)
  } else if (Array.isArray(n2.children)) {
  	if (Array.isArray(n1.children)) {
      const oldChildren = n1.children
      const newChildren = n2.children
      
      // 存储整个寻找过程中遇到的最大索引值
      let lastIndex = 0
      for (let i = 0; i < newChildren.length; i++) {
      	const newVNode = newChildren[i]
        for (let j = 0; j < oldChildren.length; j++) {
        	const oldVNode = oldChildren[j]
          if (newVNode.key === oldVNode.key) {
          	patch(oldVNode, newVNode, container)
            if (j < lastIndex) {
            	// 若当前找到的节点在旧 children 中索引小于最大索引值 lastIndex，说明该节点对应的真实 DOM 需要移动
              // 先获取 newVNode 的前一个 vnode，即 prevVNode
              const prevVNode = newChildren[i - 1]
              // 如果 prevVNode 不存在，说明当前 newVNode 是第一个节点，不需要移动
              if (prevVNode) {
              	// 由于我们需要将 newVNode 对应的真实 DOM 移动到 prevVNode 所对应真实 DOM 后，所以需要获取 prevVNode 所对应真实 DOM 的下一个兄弟节点，并将其作为锚点
                const anchor = prevVNode.el.nextSibling
                // 调用渲染器选项中的 insert 方法将 newVNode 对应的真实 DOM 插入到锚点元素前，即 prevVNode 所对应真实 DOM 后
                insert(newVNode.el, container, anchor)
              }
            } else {
            	// 若当前找到的节点在旧 children 中索引小于最大索引值 lastIndex，则更新 lastIndex
              lastIndex = j
            }
            break
          }
        }
      }
    } else {
      setElementText(container, '')
      n2.children.forEach(c => patch(null, c, comtainer))
    }
  } else { 
    if (Array.isArray(n1.children)) {
      n1.children.forEach(c => unmount(c))
    } else if (typeof n1.children === 'string') {
    	setElementText(container, '')
    }
  }
}
```
# 添加新元素 & 移除不存在的元素
修改 patch、mountElement、patchChildren 函数：
```
function patchChildren(n1, n2, container) {
	if (typeof n2.children === 'string') {
    if (Array.isArray(n1.children)) {
    	n1.children.forEach((c) => c.unmount(c))
    }
    setElementText(container, n2.children)
  } else if (Array.isArray(n2.children)) {
  	if (Array.isArray(n1.children)) {
      const oldChildren = n1.children
      const newChildren = n2.children
      
      let lastIndex = 0
      for (let i = 0; i < newChildren.length; i++) {
      	const newVNode = newChildren[i]
        let j = 0
        // 在第一层循环中定义变量 find，代表是否在旧的一组子节点中找到可复用的节点；初始值为 false，代表没找到
        let find = false
        for (j; j < oldChildren.length; j++) {
        	const oldVNode = oldChildren[j]
          if (newVNode.key === oldVNode.key) {
            // 一旦找到可复用的节点，则将变量 find 的值设为 true
            find = true
          	patch(oldVNode, newVNode, container)
            if (j < lastIndex) {
              const prevVNode = newChildren[i - 1]
              if (prevVNode) {
                const anchor = prevVNode.el.nextSibling
                insert(newVNode.el, container, anchor)
              }
            } else {
              lastIndex = j
            }
            break
          }
        }
        // 功能：移除不存在的元素（旧的一组子节点中有，新的一组子节点中无）
        // 上一步的更新操作完成后，遍历旧的一组子节点
        for (let i = 0; i < oldChildren.length; i++) {
        	const oldVNode = oldChildren[i]
          // 拿旧子节点 oldVNode 去新的一组子节点中寻找具有相同 key 值的节点
          const has = newChildren.find(vnode => vnode.key === oldVNode.key)
          if (!has) {
          	// 没有找到，说明需要调用 unmount 函数删除该节点
            unmount(oldVNode)
          }
        }
        
        // 功能：添加新元素
        // 如果运行到这，find 仍为 false，说明当前 newVNode 没有在旧的一组子节点中找到可复用的节点，需要挂载
        if (!find) {
        	// 为了将节点挂载到正确位置，先获取锚点元素
          const prevVNode = newChildren[i - 1]
          let anchor = null
          if (prevVNode) {
          	// 如果有前一个 vnode 节点，则使用它的下一个兄弟节点作为锚点元素
            anchor = prevVNode.el.nextSibling
          } else {
            // 如果没有前一个 vnode 节点，说明即将挂载到新节点是第一个子节点，这时使用容器元素的 firstChild 作为锚点
          	anchor = container.firstChild
          }
          // 挂载 newNVode
          patch(null, newNVode, container, anchor) // 将锚点元素作为 patch 函数的第四个参数（新增），因此需要完善 patch 函数
        }
      }
    } else {
      setElementText(container, '')
      n2.children.forEach(c => patch(null, c, comtainer))
    }
  } else { 
    if (Array.isArray(n1.children)) {
      n1.children.forEach(c => unmount(c))
    } else if (typeof n1.children === 'string') {
    	setElementText(container, '')
    }
  }
}

// patch 函数需要接收第四个参数，即锚点元素
function patch(n1, n2, container, anchor) {
  if (n1 && n1.type !== n2.type) {
    unmount(n1)
    n1 = null
  }

  const { type } = n2
  
  if (typeof type === 'string') {
  	if (!n1) {
      // 挂载时将锚点元素作为第三个参数传递给 mountElement 函数
  		mountElement(n2, container, anchor)
    } else {
      patchElement(n1, n2)
    }	
  } else if (typeof type === Text) {
    if (!n1) {
      const el = n2.el = createText(n2.children)
      insert(el, container)
    } else {
      const el = n2.el = n1.el
      if (n2.children !== n1.children) {
        setText(el, n2.children)
      }
    }
  } else if (type === Fragment) { 
    if (!n1) {
      n2.children.forEach(c => patch(null, c, container))
    } else {
      patchChildren(n1, n2, container)
    }
  }
}

// mountElement 函数需要增加第三个参数，即锚点元素
function mountElement(vnode, container) {
  const el = createElement(vnode.type)
  
  if (typeof vnode.children === 'string') {
    setElementText(el, vnode.children)
  } else if (Array.isArray(vnode.children)) {
    vnode.children.forEach(child => {
    	patch(null, child, el)
    })
  }
  
  if (vnode.props) {
  	for (const key in vnode.props) {
      el.setAttribute(key, vnode.props[key])
    }
  }
  
  // 插入节点时，将锚点元素透传给 insert 函数
  insert(el, container)
}
```
