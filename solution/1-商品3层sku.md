# Solution：商品三次sku优化

下面再来回顾下本文的要点：

1. 本文要实现的需求是一个商品的三层sku选项。
2. 当用户选择了两层后，第三层选项应该自动计算出哪些能卖，哪些不能卖。
3. 鉴于后端API返回选项和商品间没有直接的对应关系，为了找出能卖还是不能卖，我们需要遍历所有商品。
4. 当总商品数量不多的时候，所有商品遍历可能不会产生明显的性能问题。
5. 但是当选项增加到三层，商品数量的增加是指数级的，性能问题就会显现出来。
6. 对于`O(n³)`这种写代码时就能预见的性能问题，我们不用等着报BUG了才处理，而是开发时直接就解决了。
7. 本例要解决的是一个查找问题，所以我想到了建一颗树，直接将`O(n³)`的复杂度降到了`O(1)`。
8. 但是一颗树并不能覆盖所有的用户操作，要覆盖所有的用户操作需要6棵树。
9. 出于偷懒的目的，我跟产品经理商量，调整了需求和交互砍掉了5颗树。真实原因是树太多了，会占用更多的内存空间，也不好维护。有时候适当的调整需求和交互也可以达到优化性能的效果，性能优化可以将交互和技术结合起来思考。
10. 这个树的搜索模块可以单独封装成一个类，外部使用者，不需要知道细节，直接调用接口查找就行。
11. 前端会点数据结构还是有用的，本文这种场景下还很有必要。

## 一、背景介绍

背景：电商，商品详情页，需要支持三层sku选项。

> - `sku 属性`(会影响到库存和价格的属性, 又叫销售属性，一般是一对多、多对多的关系) 例如：容量(128G/256G/512G/1T)、颜色(暗夜紫/远峰蓝/玫瑰金)
> - `spu 属性`(不会影响到库存和价格的属性, 又叫关键属性，一般是一对一的关系) 例如：毛重(267.00g)、产地(中国大陆)

分析：目前国内商品中没有发现三层sku选项的，最多只有两侧。究其原因：
（1）这可能是个伪命题 — 一般两层sku选项足够覆盖所有商品了，再多的一次选项可以向下增加到前两层选项中，或者向上拆分为更多的商品类型；
（2）有性能问题 — 例如 两层sku，每层11个选项，用户每次进行选择是需要进行 O(n²)  `11*11=121` 个商品，需要遍历121次。如果增加到 三层sku，每次11个选项，则是 O(n³) `11*11*11=1331`个商品，遍历1331次 需要 `220ms`，用户会感觉到明显的卡顿。

## 二、解决方案

将原有的数组数据结构，构建像下面这样一颗树，可以将时间复杂度降到O(1)：

![640.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/32fb983e52174741aef2f3ff99911d10~tplv-k3u1fbpfcp-watermark.image?)

有了上面这个数据结构，我们要查找红色的39码直接取值
`tree["颜色：红色"]["尺码：39"]` 就行了，这个复杂度瞬间就变为O(1)了。

问题：
再仔细看下我们构建出来的数据结构，层级关系是固定的，第一层是颜色，第二层是尺码，第三层是性别。
因此，**用户必须严格按照我们构建的树形结构的顺序进行选择**，即 颜色->尺码->性别 的顺序，然后我们看看性别这里哪个该灰掉。**如果他不按照这个顺序，那我们就无法获取对应商品的属性**，比如他先选了性别男，然后选尺码40，这时候我们应该计算最后一个层级颜色哪些该灰掉。但是使用上面这个结构我们是算不出来的，**因为我们并没有`tree["性别：男"]["尺码：40"]`这个对象**。

解决方案一：我们没有`性别-尺码-颜色`这种顺序的树，那我们就建一颗。按照三个变量的一个全排列，就是6棵树。但是这似乎并非是最佳的选择，因为构建树的过程也是比较消耗性能的，而且并非没有其他方案可替代。

解决方案二：**给一个默认值，不提供取消功能，只能切换选项**。如果提供取消功能，他将我们提供的颜色-尺码-性别默认选项取消掉，又可以选成性别-尺码-颜色了。不提供取消功能，只能通过选择其他选项来切换，只能从红色换成白色，而不能取消红色，其他的一样。这样我们就能永远保证颜色-尺码-性别这个顺序，用户操作只是只是每个层级选中的值不一样，层级顺序并不会变化，我们的查找树就一直有效了。而且我发现某些购物网站也不能取消选项，不知道他们是不是也遇到了类似的问题。

## 三、性能优化

前面的方案我们解决了查找的性能问题，但是引入了一个新问题，那就是需要创建这颗查找树。创建这颗查找树还是需要对商品列表进行一次遍历，这是不可避免的，为了更顺滑的用户体验，**我们应该尽量将这个构建树的过程隐藏在用户感知不到的地方**。我这里是将它整合到了商品详情页的加载状态中，用户点击进入商品详情页，我们要去API取数据，不可避免的会有一个加载状态，会转个圈什么的。我将这个遍历过程也做到了这个转圈中，当API数据返回，并且查找树创建完成后，转圈才会结束。这在理论上会延长转圈的时间，但是本地的遍历再慢也会比网络请求快点，所以用户感知并不明显。当转圈结束后，所有数据都准备就绪了，用户操作都是O(1)的复杂度，做到了真正的丝般顺滑。



```JavaScript
class VariationSearchMap {
  constructor(apiData) {
    this.tree = this.buildTree(apiData);
  }

  // 这就是前面那个构造树的方法
  buildTree(apiData) {
    const tree = {};
    const { variations, products } = apiData;

    // 先用variations将树形结构构建出来，叶子节点默认值为null
    addNode(tree, 0);
    function addNode(root, deep) {
      const variationName = variations[deep].name;
      const variationValues = variations[deep].values;

      for (let i = 0; i < variationValues.length; i++) {
        const nodeName = `${variationName}：${variationValues[i].name}`;
        if (deep === variations.length - 1) {
          root[nodeName] = null;
        } else {
          root[nodeName] = {};
          addNode(root[nodeName], deep + 1);
        }
      }
    }

    // 然后遍历一次products给树的叶子节点填上值
    for (let i = 0; i < products.length; i++) {
      const product = products[i];
      const { variationMappings } = product;
      const level1Name = `${variationMappings[0].name}：${variationMappings[0].value}`;
      const level2Name = `${variationMappings[1].name}：${variationMappings[1].value}`;
      const level3Name = `${variationMappings[2].name}：${variationMappings[2].value}`;
      tree[level1Name][level2Name][level3Name] = product;
    }

    // 最后返回构建好的树
    return tree;
  }

  // 添加一个方法来搜索商品，参数结构和API数据的variationMappings一样
  findProductByVariationMappings(variationMappings) {
    const level1Name = `${variationMappings[0].name}：${variationMappings[0].value}`;
    const level2Name = `${variationMappings[1].name}：${variationMappings[1].value}`;
    const level3Name = `${variationMappings[2].name}：${variationMappings[2].value}`;

    const product = this.tree[level1Name][level2Name][level3Name];

    return product;
  }
}

```

然后使用的时候直接new一下就行：

```JavaScript
const variationSearchMap = new VariationSearchMap(apiData);    // new一个实例出来

// 然后就可以用这个实例进行搜索了
const searchCriteria = [
  { name: '颜色', value: '红色' },
  { name: '尺码', value: '40' },
  { name: '性别', value: '女' }
];
const matchedProduct = variationSearchMap.findProductByVariationMappings(searchCriteria);
console.log('matchedProduct', matchedProduct);    // { productId: 8 }
```

本文可运行的示例代码已经上传GitHub，大家可以拿下来玩玩：https://github.com/dennis-jiang/Front-End-Knowledges/tree/master/Examples/DataStructureAndAlgorithm/OptimizeVariations

> [速度提高几百倍，记一次数据结构在实际工作中的运用](https://mp.weixin.qq.com/s/q5YDEmohyrtQ_teS0Ws7Fg)
