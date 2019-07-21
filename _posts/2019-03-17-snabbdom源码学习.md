---
layout: post
title:  "snabbdom源码学习"
date:   2019-03-17 21:27:01 +0800
categories: js
tag: vue
---

* content
{:toc}

https://github.com/snabbdom/snabbdom#examples

**注意attention:** [转载学习来源--snabbdom](https://github.com/snabbdom/snabbdom)  

**注意attention:** [转载学习来源--vue2源码学习开胃菜——snabbdom源码学习（一）](https://segmentfault.com/a/1190000009017324)  
**注意attention:** [转载学习来源--vue2源码学习开胃菜——snabbdom源码学习（二）](https://segmentfault.com/a/1190000009017349)  

**注意attention:** [转载学习来源--深入Vue2.x的虚拟DOM diff原理](https://blog.csdn.net/M6i37JK/article/details/78140159)  


snabbdom源码学习			{#snabbom}
====================================

我们知道，当我们希望实现一个具有复杂状态的界面时，如果我们在每个可能发生变化的组件上都绑定事件，绑定字段数据，那么很快由于状态太多，我们需要维护的事件和字段将会越来越多，代码也会越来越复杂，于是，我们想我们可不可以将视图和状态分开来，只要视图发生变化，对应状态也发生变化，然后状态变化，我们再重绘整个视图就好了。这样的想法虽好，但是代价太高了，于是我们又想，能不能只更新状态发生变化的视图？于是virtual-dom应运而生，状态变化先反馈到vdom上，vdom在找到最小更新视图，最后批量更新到真实DOM上，从而达到性能的提升。  

除此之外，从移植性上看，virtual-dom还对真实dom做了一次抽象，这意味着virtual-dom对应
的可以不是浏览器的dom，而是不同设备的组件，极大的方便了多平台的使用。  

使用	{#snabbom_use}
------------------------------------

```javascript
var snabbdom = require('snabbdom');
var patch = snabbdom.init([ // Init patch function with chosen modules
  require('snabbdom/modules/class').default, // makes it easy to toggle classes
  require('snabbdom/modules/props').default, // for setting properties on DOM elements
  require('snabbdom/modules/style').default, // handles styling on elements with support for animations
  require('snabbdom/modules/eventlisteners').default, // attaches event listeners
]);
var h = require('snabbdom/h').default; // helper function for creating vnodes

var container = document.getElementById('container');

var vnode = h('div#container.two.classes', {on: {click: someFn}}, [
  h('span', {style: {fontWeight: 'bold'}}, 'This is bold'),
  ' and this is just normal text',
  h('a', {props: {href: '/foo'}}, 'I\'ll take you places!')
]);
// Patch into empty DOM element – this modifies the DOM as a side effect
patch(container, vnode);

var newVnode = h('div#container.two.classes', {on: {click: anotherEventHandler}}, [
  h('span', {style: {fontWeight: 'normal', fontStyle: 'italic'}}, 'This is now italic type'),
  ' and this is still just normal text',
  h('a', {props: {href: '/bar'}}, 'I\'ll take you places!')
]);
// Second `patch` invocation
patch(vnode, newVnode); // Snabbdom efficiently updates the old view to the new state

```

vnode   	{#snabbom_vnode}
------------------------------------

```javascript
export interface VNode {
  sel: string | undefined; // 对应的是选择器,如'div','div#a','div#a.b.c'的形式
  data: VNodeData | undefined; // 对应的是vnode绑定的数据，可以有以下类型：attribute、props、eventlistner、class、dataset、hook
  children: Array<VNode | string> | undefined; // 子元素数组
  elm: Node | undefined; // 里面存储着对对应的真实dom element的引用
  text: string | undefined; // 文本，代表该节点中的文本内容
  key: Key | undefined; // 用于不同vnode之间的比对
}
```

hook   	{#snabbom_hook}
------------------------------------

+ create => style,class,dataset,eventlistener,props,hero  
+ update => style,class,dataset,eventlistener,props,hero  
+ remove => style  
+ destory => eventlistener,style,hero  
+ pre => hero  
+ post => hero  


init   	{#snabbom_init}
------------------------------------

```javascript
function isUndef(s: any): boolean { return s === undefined; }
function isDef(s: any): boolean { return s !== undefined; }

type VNodeQueue = Array<VNode>;

const emptyNode = vnode('', {}, [], undefined, undefined);

//这个函数主要用于比较oldvnode与vnode同层次节点的比较，如果同层次节点的key和sel都相同
// 我们就可以保留这个节点，否则直接替换节点
function sameVnode(vnode1: VNode, vnode2: VNode): boolean {
  return vnode1.key === vnode2.key && vnode1.sel === vnode2.sel;
}

function isVnode(vnode: any): vnode is VNode {
  return vnode.sel !== undefined;
}

type KeyToIndexMap = {[key: string]: number};

type ArraysOf<T> = {
  [K in keyof T]: (T[K])[];
}

type ModuleHooks = ArraysOf<Module>;
//获取oldvnode数组中 key对位置的映射
function createKeyToOldIdx(children: Array<VNode>, beginIdx: number, endIdx: number): KeyToIndexMap {
  let i: number, map: KeyToIndexMap = {}, key: Key | undefined, ch;
  for (i = beginIdx; i <= endIdx; ++i) {
    ch = children[i];
    if (ch != null) {
      key = ch.key;
      if (key !== undefined) map[key] = i;
    }
  }
  return map;
}

const hooks: (keyof Module)[] = ['create', 'update', 'remove', 'destroy', 'pre', 'post'];

export {h} from './h';
export {thunk} from './thunk';

export function init(modules: Array<Partial<Module>>, domApi?: DOMAPI) {
  let i: number, j: number, cbs = ({} as ModuleHooks);
  // domApi的扩展设置，方便兼容性扩展
  const api: DOMAPI = domApi !== undefined ? domApi : htmlDomApi;

  // 注册钩子的回调，在发生状态变更时，触发对应属性变更
  // 很重要的方法，在此处注册对于module目录里面各种文件对于VNodeData的处理
  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = [];
    for (j = 0; j < modules.length; ++j) {
      const hook = modules[j][hooks[i]];
      if (hook !== undefined) {
        (cbs[hooks[i]] as Array<any>).push(hook);
      }
    }
  }

  function emptyNodeAt(elm: Element) {
    const id = elm.id ? '#' + elm.id : '';
    const c = elm.className ? '.' + elm.className.split(' ').join('.') : '';
    return vnode(api.tagName(elm).toLowerCase() + id + c, {}, [], undefined, elm);
  }
  // 对remove钩子回调操作的计数功能
  function createRmCb(childElm: Node, listeners: number) {
    return function rmCb() {
      if (--listeners === 0) {
        const parent = api.parentNode(childElm);
        api.removeChild(parent, childElm);
      }
    };
  }

  function createElm(vnode: VNode, insertedVnodeQueue: VNodeQueue): Node {
    let i: any, data = vnode.data;
    if (data !== undefined) {
        // 当节点上存在hook而且hook中有init钩子时，先调用init回调，对刚创建的vnode进行处理
      if (isDef(i = data.hook) && isDef(i = i.init)) {
        i(vnode);
        data = vnode.data;
      }
    }
    let children = vnode.children, sel = vnode.sel;
    // 当sel == "!"的时候表示这个vnode就是一个comment
    if (sel === '!') {
      if (isUndef(vnode.text)) {
        vnode.text = '';
      }
      vnode.elm = api.createComment(vnode.text as string);
    } else if (sel !== undefined) {
      // Parse selector 这么一段就是为了从sel中获得tag值,id值,class值
      const hashIdx = sel.indexOf('#');
      const dotIdx = sel.indexOf('.', hashIdx);
      const hash = hashIdx > 0 ? hashIdx : sel.length;
      const dot = dotIdx > 0 ? dotIdx : sel.length;
      const tag = hashIdx !== -1 || dotIdx !== -1 ? sel.slice(0, Math.min(hash, dot)) : sel;
      const elm = vnode.elm = isDef(data) && isDef(i = (data as VNodeData).ns) ? api.createElementNS(i, tag): api.createElement(tag);
      if (hash < dot) elm.setAttribute('id', sel.slice(hash + 1, dot));
      if (dotIdx > 0) elm.setAttribute('class', sel.slice(dot + 1).replace(/\./g, ' '));
      // 调用create钩子，利用cbs里面的module hook函数进行VNodeData挂载到dom元素ele上
      for (i = 0; i < cbs.create.length; ++i) cbs.create[i](emptyNode, vnode);
      // 进行dom子元素的生成和VNodeData挂载
      if (is.array(children)) {
        for (i = 0; i < children.length; ++i) {
          const ch = children[i];
          if (ch != null) {
            // 深度遍历
            api.appendChild(elm, createElm(ch as VNode, insertedVnodeQueue));
          }
        }
      } else if (is.primitive(vnode.text)) {
        api.appendChild(elm, api.createTextNode(vnode.text));
      }
      i = (vnode.data as VNodeData).hook; // Reuse variable
      if (isDef(i)) {
        if (i.create) i.create(emptyNode, vnode);
        // ？如果有insert钩子，则推进insertedVnodeQueue中作记录，从而实现批量插入触发insert回调 
        if (i.insert) insertedVnodeQueue.push(vnode);
      }
    } else {
      vnode.elm = api.createTextNode(vnode.text as string);
    }
    return vnode.elm;
  }

  function addVnodes(parentElm: Node,
                     before: Node | null,
                     vnodes: Array<VNode>,
                     startIdx: number,
                     endIdx: number,
                     insertedVnodeQueue: VNodeQueue) {
    for (; startIdx <= endIdx; ++startIdx) {
      const ch = vnodes[startIdx];
      if (ch != null) {
        api.insertBefore(parentElm, createElm(ch, insertedVnodeQueue), before);
      }
    }
  }

  function invokeDestroyHook(vnode: VNode) {
    let i: any, j: number, data = vnode.data;
    if (data !== undefined) {
      if (isDef(i = data.hook) && isDef(i = i.destroy)) i(vnode);
      for (i = 0; i < cbs.destroy.length; ++i) cbs.destroy[i](vnode);
      if (vnode.children !== undefined) {
        for (j = 0; j < vnode.children.length; ++j) {
          i = vnode.children[j];
          if (i != null && typeof i !== "string") {
            invokeDestroyHook(i);
          }
        }
      }
    }
  }

  function removeVnodes(parentElm: Node,
                        vnodes: Array<VNode>,
                        startIdx: number,
                        endIdx: number): void {
    for (; startIdx <= endIdx; ++startIdx) {
      let i: any, listeners: number, rm: () => void, ch = vnodes[startIdx];
      if (ch != null) {
        if (isDef(ch.sel)) {
          invokeDestroyHook(ch);
          //对全局remove钩子进行计数
          listeners = cbs.remove.length + 1;
          rm = createRmCb(ch.elm as Node, listeners);
          //调用全局remove回调函数，并每次减少一个remove钩子计数（即调用remove.length次rm）
          for (i = 0; i < cbs.remove.length; ++i) cbs.remove[i](ch, rm);
          //调用内部vnode.data.hook中的remove钩子（只有一个）
          if (isDef(i = ch.data) && isDef(i = i.hook) && isDef(i = i.remove)) {
            i(ch, rm);
          } else {
            //如果没有内部remove钩子，需要调用rm，确保能够remove节点
            rm();
          }
        } else { // Text node
          api.removeChild(parentElm, ch.elm as Node);
        }
      }
    }
  }

  function updateChildren(parentElm: Node,
                          oldCh: Array<VNode>,
                          newCh: Array<VNode>,
                          insertedVnodeQueue: VNodeQueue) {
    let oldStartIdx = 0, newStartIdx = 0;
    let oldEndIdx = oldCh.length - 1;
    let oldStartVnode = oldCh[0];
    let oldEndVnode = oldCh[oldEndIdx];
    let newEndIdx = newCh.length - 1;
    let newStartVnode = newCh[0];
    let newEndVnode = newCh[newEndIdx];
    let oldKeyToIdx: any;
    let idxInOld: number;
    let elmToMove: VNode;
    let before: any;

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (oldStartVnode == null) {
        oldStartVnode = oldCh[++oldStartIdx]; // Vnode might have been moved left
      } else if (oldEndVnode == null) {
        oldEndVnode = oldCh[--oldEndIdx];
      } else if (newStartVnode == null) {
        newStartVnode = newCh[++newStartIdx];
      } else if (newEndVnode == null) {
        newEndVnode = newCh[--newEndIdx];
        // 如果旧头索引节点和新头索引节点相同，对旧头索引节点和新头索引节点进行diff更新， 从而达到复用节点效果
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue);
        oldStartVnode = oldCh[++oldStartIdx]; //旧头索引向后
        newStartVnode = newCh[++newStartIdx]; //新头索引向后
        //如果旧尾索引节点和新尾索引节点相似，可以复用
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue);
        oldEndVnode = oldCh[--oldEndIdx];
        newEndVnode = newCh[--newEndIdx];
        // 如果旧头索引节点和新头索引节点相似，可以通过移动来复用
        // 如旧节点为【5,1,2,3,4】，新节点为【1,2,3,4,5】，如果缺乏这种判断，意味着
        // 那样需要先将5->1,1->2,2->3,3->4,4->5五次删除插入操作，即使是有了key-index来复用，
        // 也会出现【5,1,2,3,4】->【1,5,2,3,4】->【1,2,5,3,4】->【1,2,3,5,4】->【1,2,3,4,5】
        // 共4次操作，如果有了这种判断，我们只需要将5插入到最后一次操作即可
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue);
        api.insertBefore(parentElm, oldStartVnode.elm as Node, api.nextSibling(oldEndVnode.elm as Node));
        oldStartVnode = oldCh[++oldStartIdx];
        newEndVnode = newCh[--newEndIdx];
        //原理与上面相同
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue);
        api.insertBefore(parentElm, oldEndVnode.elm as Node, oldStartVnode.elm as Node);
        oldEndVnode = oldCh[--oldEndIdx];
        newStartVnode = newCh[++newStartIdx];
        //如果上面的判断都不通过，我们就需要key-index表来达到最大程度复用了
      } else {
        if (oldKeyToIdx === undefined) {
          oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx);
        }
        idxInOld = oldKeyToIdx[newStartVnode.key as string];
        if (isUndef(idxInOld)) { // New element
          //如果新节点在旧节点中不存在，我们将它插入到旧头索引节点前，然后新头索引向后
          api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm as Node);
          newStartVnode = newCh[++newStartIdx];
        } else {
          elmToMove = oldCh[idxInOld];
          // 虽然 key 相同了，但是 seletor 不相同，需要调用 createElm 来创建新的 dom 节点
          if (elmToMove.sel !== newStartVnode.sel) {
            api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm as Node);
          } else {
            // 否则调用 patchVnode 对旧 vnode 做更新
            patchVnode(elmToMove, newStartVnode, insertedVnodeQueue);
            //然后将旧节点组中对应节点设置为undefined,代表已经遍历过了，不在遍历，否则可能存在重复插入的问题
            oldCh[idxInOld] = undefined as any;
            //插入到旧头索引节点之前
            api.insertBefore(parentElm, (elmToMove.elm as Node), oldStartVnode.elm as Node);
          }
          //新头索引向后
          newStartVnode = newCh[++newStartIdx];
        }
      }
    }
    // 判断是否相遇=的那一次
    if (oldStartIdx <= oldEndIdx || newStartIdx <= newEndIdx) {
      //当旧头索引大于旧尾索引时，代表旧节点组已经遍历完，将剩余的新Vnode添加到最后一个新节点的位置后
      if (oldStartIdx > oldEndIdx) {
        before = newCh[newEndIdx+1] == null ? null : newCh[newEndIdx+1].elm;
        addVnodes(parentElm, before, newCh, newStartIdx, newEndIdx, insertedVnodeQueue);
        //如果新节点组先遍历完，那么代表旧节点组中剩余节点都不需要，所以直接删除
      } else {
        removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
      }
    }
  }

  function patchVnode(oldVnode: VNode, vnode: VNode, insertedVnodeQueue: VNodeQueue) {
    let i: any, hook: any;
    if (isDef(i = vnode.data) && isDef(hook = i.hook) && isDef(i = hook.prepatch)) {
      i(oldVnode, vnode);
    }
    // 设置新vnode的elm为旧的elm，方便module hook
    const elm = vnode.elm = (oldVnode.elm as Node);
    let oldCh = oldVnode.children;
    let ch = vnode.children;
    if (oldVnode === vnode) return;
    // 我们先patch vnode,patch的方法就是先调用全局的update hook
    // 然后调用vnode.data定义的update hook
    if (vnode.data !== undefined) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode);
      i = vnode.data.hook;
      if (isDef(i) && isDef(i = i.update)) i(oldVnode, vnode);
    }
    // patch两个vnode的text和children
    // 查看vnode.text定义
    // vdom中规定,具有text属性的vnode不应该具备children
    // 对于<p>foo:<b>123</b></p>的良好写法是
    // h('p', [ 'foo:', h('b', '123')]), 而非
    // h('p', 'foo:', [h('b', '123')])
    //如果vnode不是text节点
    if (isUndef(vnode.text)) {
      //如果vnode和oldVnode都有子节点
      if (isDef(oldCh) && isDef(ch)) {
        //当Vnode和oldvnode的子节点不同时，调用updatechilren函数，diff子节点
        if (oldCh !== ch) updateChildren(elm, oldCh as Array<VNode>, ch as Array<VNode>, insertedVnodeQueue);
      //如果vnode有子节点，oldvnode没子节点
      } else if (isDef(ch)) {
        //oldvnode是text节点，则将elm的text清除
        if (isDef(oldVnode.text)) api.setTextContent(elm, '');
        //并添加vnode的children
        addVnodes(elm, null, ch as Array<VNode>, 0, (ch as Array<VNode>).length - 1, insertedVnodeQueue);
        //如果oldvnode有children，而vnode没children，则移除elm的children
      } else if (isDef(oldCh)) {
        removeVnodes(elm, oldCh as Array<VNode>, 0, (oldCh as Array<VNode>).length - 1);
        //如果vnode和oldvnode都没chidlren，且vnode没text，则删除oldvnode的text
      } else if (isDef(oldVnode.text)) {
        api.setTextContent(elm, '');
      }
    //如果oldvnode的text和vnode的text不同，则更新为vnode的text
    } else if (oldVnode.text !== vnode.text) {
      if (isDef(oldCh)) {
        removeVnodes(elm, oldCh as Array<VNode>, 0, (oldCh as Array<VNode>).length - 1);
      }
      api.setTextContent(elm, vnode.text as string);
    }
    if (isDef(hook) && isDef(i = hook.postpatch)) {
      i(oldVnode, vnode);
    }
  }

  return function patch(oldVnode: VNode | Element, vnode: VNode): VNode {
    let i: number, elm: Node, parent: Node;
    // 记录被插入的vnode队列，用于批触发insert
    const insertedVnodeQueue: VNodeQueue = [];
    for (i = 0; i < cbs.pre.length; ++i) cbs.pre[i]();
    // 如果oldVnode不是vnode(在第一次调用时,oldVnode是dom element)
    // 那么用emptyNodeAt函数来将其包装为vnode
    if (!isVnode(oldVnode)) {
      oldVnode = emptyNodeAt(oldVnode);
    }
    // sameVnode是上述“值不值得patch”的核心
    // sameVnode实现很简单,查看两个vnode的key与sel是否分别相同
    // ()=>{vnode1.key === vnode2.key && vnode1.sel === vnode2.
    // 比较语义不同的结构没有意义,比如diff一个'div'和'span'
    // 而应该移除div,根据span vnode插入新的span
    // diff两个key不相同的vnode同样没有意义
    // 指定key就是为了区分element
    // 对于不同key的element,不应该去根据newVnode来改变oldVnode的数据
    // 而应该移除不再oldVnode,添加newVnode
    if (sameVnode(oldVnode, vnode)) {
      // oldVnode与vnode的sel和key分别相同,那么这两个vnode值得去比较
      // patchVnode根据vnode来更新oldVnode
      patchVnode(oldVnode, vnode, insertedVnodeQueue);
    } else {
      // 不值得去patch的,我们就暴力点
      // 移除oldVnode,根据newVnode创建elm,并添加至parent中
      elm = oldVnode.elm as Node;
      parent = api.parentNode(elm);

      createElm(vnode, insertedVnodeQueue);

      if (parent !== null) {
        api.insertBefore(parent, vnode.elm as Node, api.nextSibling(elm));
        removeVnodes(parent, [oldVnode], 0, 0);
      }
    }

    for (i = 0; i < insertedVnodeQueue.length; ++i) {
      (((insertedVnodeQueue[i].data as VNodeData).hook as Hooks).insert as any)(insertedVnodeQueue[i]);
    }
    for (i = 0; i < cbs.post.length; ++i) cbs.post[i]();
    return vnode;
  };
}



```





