---
layout: post
title:  "js实现PriorityQueue类"
date:   2022-09-16 15:08:00 +0800
categories: data-construct queue priority-heap
---

js原生是没有提供这种复杂数据结构的（默默的羡慕一下java的世界）。
为什么需要这种结构的呢？这种结构的优势又是什么呢？

## Queue vs PriorityQueue
我们对这个词都不陌生，女王，哦，不，是：队列。

队列的特点就是“先进先出”，进、出的时间效率都是O(1)；在js世界中，使用朴素的Array结构就可以模拟。那如果进、出都是有条件的，我们会怎么处理呢？
比如，一堆学生在一个房间里，现在需要一个一个按需叫他们出来谈话，我们现在定一个规则就是O(1)时间内，按照学生的身高由高到低出来；而在谈话的过程中呢，又会有一些新的学生进入教室，那我们就需要通过某种结构进行管理，某种方法进行插入，以达到我们能O(1)效率找到当前学生中最高的目标。

列举的这个场景，比较的维度还是线性的：身高（数值对比）。数据结构中的大顶堆可以解决这个问题。
那如果现在排序的规则发生变化了呢？诸如，按照学生家庭住址的经纬度（二次元关系，非线形关系），我们又该如何处理呢？此时堆中的value不是单纯的值，而可能是任何结构的数据。

我们可以抽取出所谓的排序规则为：<span style="background-color:#feff0078;font-weight:bold;">comparetor</span>。

在构建堆的过程中都使用comparetor对节点进行位置的确定。


## Java中的PriorityQueue
论原厂装备来，无疑java这种天生富贵的静态语言自带的武器简直爆表。

官方文档中对此数据结构的解释：[链接](https://docs.oracle.com/en/java/javase/19/docs/api/java.base/java/util/PriorityQueue.html)


<img src="https://github.com/qianghe/blogs/blob/main/imgs/java-priorityQueue.png?raw=true"  style="zoom:40%;" />


<img src="https://github.com/qianghe/blogs/blob/main/imgs/java-priorityQueue-class.png?raw=true"  style="zoom:30%;" />

PriorityQueue类基于priority heap，并且其实现了Collection类和Iterator类的可选方法。
从性能的角度来看：
offer、poll、add（进出队列的函数）的时间效率是O(logn)；
contain、remove(object)为线性时间O(n)；
peek（获取元素）等操作为常量时间O(1)。


## js实现PriorityQueue

需要实现的几个方法如下：

<img src="https://github.com/qianghe/blogs/blob/main/imgs/js-priorityQueue-implement.png?raw=true"  style="zoom:30%;" />

实现基于的数据结构也是堆结构（用array模拟二叉树结构）。

```javascript
// 大顶堆
const defaultComparetor = (a, b) => b - a

class PriorityQueue {
	constructor(comparetor) {
		this.queue = []
		this.comparetor = comparetor
	}

	_swap(x, y) {
		const tmp = this. queue[x]
		this.queue[x] = this.queue[y]
		this.queue[y] = tmp
	}

	_downCheck(loc) {
		const [lLoc, rLoc] = [2 * loc + 1, 2 * loc + 2]
		
		if (this.queue[lLoc] !== undefined) {
			if (this.comparetor(this.queue[loc], this.queue[lLoc]) > 0) {
				this._swap(loc, lLoc)
				this._downCheck(lLoc)
			}
		}

		if (this.queue[rLoc] !== undefined) {
			if (this.comparetor(this.queue[loc], this.queue[rLoc]) > 0) {
				this._swap(loc, rLoc)
				this._downCheck(rLoc)
			}
		}
	}

	_upCheck(cur) {
		while (cur > 0) {
			const parent = (cur - (cur % 2 === 0 ? 2 : 1)) / 2
			if (this.comparetor(this.queue[parent], this.queue[cur]) > 0) {
				this._swap(parent, cur)
				cur = parent
			} else {
				break
			}
		}
	}

	getLen() {
		return this.queue.length
	}

	isEmpty() {
		return this.getLen() === 0
	}


	// 删除指定元素
	del (x) {
		const loc = this.queue.indexOf(x)
		
		this._swap(loc, this.getLen() - 1)
		this.queue.pop()
		this._upCheck(loc)
		this._downCheck(loc)
	}

	// 插入
	add(x) {
		this.queue.push(x)
		let cur = this.getLen() - 1
		this._upCheck(cur)
	}

	// 取堆顶元素
	poll() {
		this._swap(0, this.getLen() - 1)
		const top = this.queue.pop()
		this._downCheck(0)

		return top
	}

	peek() {
		return this.queue[0]
	}
}
```



