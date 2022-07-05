---
layout: post
title:  "[Leetcode Range Module]-算法解析"
date:   2022-07-05 20:44:02 +0800
categories: leetcode design
---

## 题目

[https://leetcode.com/problems/range-module/](https://leetcode.com/problems/range-module/)


## 分析及实现

这道题从分析角度并不难，但本人还是花了几天碎片时间才通过了所有的case，就其原因还是一些边界条件没有梳理清楚。下面根据其需要实现的几个基础（query\add\remove）方法进行分析。


```javascript
var RangeModule = function() {
    this.segs = []
};
```
通过segs为维持当前的seg段落，并根据二分查找法对新的seg([a, b])进行搜索，即二分查找寻找其需要插入的segs位置。

二分查找实现：

```javascript
RangeModule.prototype._findIndex = function(num, loc) {
  let l = 0;
  let r = this.segs.length - 1

  while (l <= r) {
    const mid = ~~((l + r) / 2)
    const midNum = this.segs[mid][loc]
    if (midNum === num) return mid
    else if (midNum < num) {
      l = mid + 1
    } else {
      r = mid - 1
    }
  }

  return l
}
```

### query

寻找段落querySeg（[a, b]）。

如果寻找成功，意味着，[a, b]与当前segs中的某一个段落seg是有交集的，即：querySeg ⋂ seg === querySeg

```javascript
let leftLoc = this._findIndex(left, 0)
let rightLoc = this._findIndex(right, 1)
```

那么针对querySeg，我们通过二分查找，分别找到了leftLoc和rightLoc。（通过二分查找出的loc，包含等于的情况，因此我们需要注意此处的边界条件！）。

<img src="https://cdn.nlark.com/yuque/0/2022/jpeg/388960/1657009527633-209a3041-39fb-4722-a275-ca99b8afebb3.jpeg" />

分情况分析：
1.  排除边界（图中的情况A、B）
	
	1.1.  左边界：leftLoc === 0，seg1.left < querySeg.left ，说明超出左边界，返回false；
	
	1.2.  有边界：rightLoc === len, seg4.right < querySeg.right，说明超出右边界，返回false；

2.  修正leftLoc（如果不是等于segs[leftLoc]左边界的情况，需要移动到前一个seg）

	2.1.  leftLoc !== rightLoc（情况C），未命中同一个seg，返回false；
	
	2.2.  leftLoc === rightLoc，如果是情况D，返回false， 否则返回true。

最终实现：
```javascript
RangeModule.prototype.queryRange = function(left, right) {
    if (this._isEmtpy()) return false
    const len = this.segs.length
    let leftLoc = this._findIndex(left, 0)
    let rightLoc = this._findIndex(right, 1)
    
    if (leftLoc === 0 && this.segs[0][0] > left) return false
    if (rightLoc === len && this.segs[len - 1][1] > right) return false

    if (leftLoc === len || (leftLoc !== 0 && this.segs[leftLoc][0] !== left)) {
      leftLoc -= 1
    }

    if (leftLoc !== rightLoc) return false
    
    const seg = this.segs[leftLoc]
    return seg[0] <= left && seg[1] >= right
};
```

### addRange

插入段落insertSeg[a, b]。

let leftLoc = this._findIndex(left, 0)
let rightLoc = this._findIndex(right, 1)

<img src="https://cdn.nlark.com/yuque/0/2022/jpeg/388960/1657012289333-9860149f-34c5-4c59-8660-ab316841247d.jpeg" />

```javascript
RangeModule.prototype.addRange = function(left, right) {
  if (this._isEmtpy()) {
    this.segs.push([left, right])
    return
  }

  if (this.queryRange(left, right)) return
  const len = this.segs.length
  let leftLoc = this._findIndex(left, 0)
  
  if (leftLoc === len && this.segs[leftLoc - 1][1] < left) {
    this.segs.push([left, right])
    return
  }
  
  if (leftLoc !== 0 && this.segs[leftLoc - 1][1] >= left) {
    leftLoc -= 1
  }
  
  let rightLoc = this._findIndex(right, 1)
  if (rightLoc === 0 && right < this.segs[0][0]) {
    this.segs.unshift([left, right])
    return
  }
  if (rightLoc !== 0 && rightLoc !== len && this.segs[rightLoc][0] > right) {
    rightLoc -= 1
  }

  this.segs.splice(leftLoc, rightLoc - leftLoc + 1, [
    Math.min(left, this.segs[leftLoc][0]),
    rightLoc === len ? right : Math.max(right, this.segs[rightLoc][1])
  ])
}
```

### removeRange

删除段落removeSeg[a, b]。

```javascript
let leftLoc = this._findIndex(left, 0)
let rightLoc = this._findIndex(right, 1)
```

<img src="https://cdn.nlark.com/yuque/0/2022/jpeg/388960/1657012360910-38c5c413-eb30-4179-83db-3d6c39b0a93a.jpeg" />

情况分析：

1.  边界情况：

	1.1.  情况A，leftLoc === len，并且当前segs的右边缘seg4.right < querySeg.left，在右边界外，不需要删除；
	
	1.2.  情况B，rightLoc === 0, 并且当前segs的左边缘seg1.left > query.right，在左边界处外，不需要删除；

2.  修正leftLoc、rightLoc：

	2.1.  情况C， leftLoc === rightLoc， 并且seg3.left > left && seg3.right > right，不需要删除；
	
	2.2.  其他情况，均可正常进行移除，这里对移除需要分几种情况：
	
  a.  情况D1：leftSeg.left < left && rightSeg.right > right，这时，removeSeg在某些片段中间，因此会将这些seg删除到只剩下两个片段，分别是[leftSeg[0], left]、[right, rightSeg[1]];
  
  b.  情况D2: leftSeg.left >= left，这时我只需要考虑rightSeg被删除的状况，剩下的seg: [right, rightSeg[1]]	;
  
  c.  情况D3: right.right <= right，这时我只需要考虑leftSeg被删除的状况，剩下seg: [leftSeg[0], left]	;
  
  d.  情况D4: removeSeg完全覆盖了某些seg，只需要删除这些seg即可。

实现如下：

```javascript
RangeModule.prototype.removeRange = function(left, right) {
  if (this._isEmtpy()) return

  const len = this.segs.length
  let leftLoc = this._findIndex(left, 0)
  if (leftLoc === len && this.segs[leftLoc - 1][1] < left) return
  if (leftLoc !== 0 && this.segs[leftLoc - 1][1] >= left) {
    leftLoc -= 1
  }

  let rightLoc = this._findIndex(right, 1)
  if (rightLoc === len && this.segs[rightLoc - 1][1] < left) return
  if (rightLoc !== 0 && rightLoc !== len && this.segs[rightLoc][0] > right) {
    rightLoc -= 1
  }

  if (leftLoc === rightLoc) {
    const seg = this.segs[leftLoc]
    if (seg[0] > left && seg[0] > right) return
  }

  const leftSeg = this.segs[leftLoc]
  let rightSeg = this.segs[rightLoc]
  if (!rightSeg) {
    rightSeg = this.segs[rightLoc - 1]
    rightLoc -= 1
  }

  const adds = []
  
  if (leftSeg[0] < left && rightSeg[1] > right) {
    adds.push([leftSeg[0], left], [right, rightSeg[1]])
  } else if (leftSeg[0] >= left) {
    adds.push([right, rightSeg[1]])
  } else if (rightSeg[1] <= right) {
    adds.push([leftSeg[0], left])
  }
  const params = [leftLoc, rightLoc - leftLoc + 1]

  if (adds.length > 0) {
    params.push(...adds)
  }
  this.segs.splice(...params)
};
```

## 总结

这道题我觉得比较难处理的是对边界的考虑，包括二分查找中还包括等于的情况。但基于上面的分析，其实主要逻辑就是：

1. 处理边界；
2.  修复二分查找的index；
3.  对leftLoc、rightLoc所定位的seg，针对不同场景进行判断：
   * query判断是否为同一个seg，且是其交叉seg的子集；
   
   * add判断插入的点几插入区间范围；

   * remove判断删除的点及删除后新的seg。
