title: React-Native 样式管理方案浅谈
date: 2017-07-29 02:44:00
description:
categories: 前端开发
tags:
  - React Native
toc:
feature:
---

### StyleSheet API

* hairlineWidth

常量，定义了当前运行设备平台上的最细的宽度，但事实上`1/PixelRatio.get()`也能得到该常量。

* flatten

通过id查找样式表对应id的样式对象。

* absoluteFillObject

其实就是等价于:
`{position: "absolute", left: 0, right: 0, top: 0, bottom: 0}`

* absoluteFill

当然，其也是等价于:
`StyleSheet.create({test:{...StyleSheet.absoluteFillObject}}).test`

## StyleSheet 问题

    StyleSheet只支持`style={[StyleSheet.create({test:{flex:1}}).test]}`，却无法支持`style={{...StyleSheet.create({test:{flex:1}}).test}}`,本质上`StyleSheet.create({test:{flex:1}}).test`的类型只不过是`[object Number]`。

在其[官方文档](https://facebook.github.io/react-native/docs/stylesheet.html)中,Performance这里有一行文字：

```
It also allows to send the style only once through the bridge. All subsequent uses are going to refer an id (not implemented yet).
```

当然从性能到角度上来说，既然React-Native设计StyleSheet最根本的目的就是文档提到的：创建创建一个样式表，然后利用ID来引用样式，减少频繁创建新的样式对象。

**但是一个单级引用的样式对象，我认为这样的样式表的性能意义已经远低于其设计的易用性和可扩展性。**

### React-Native 样式管理的痛点:

1. 主题式管理
2. 混合样式优先级
3. 命名空间
4. 样式继承
5. 属性集合

github上面目能找到的比较突出的几个解决方案有：
1. [styled-components](https://github.com/styled-components/styled-components)(Web CSS增强转换器)
2. [react-native-css](https://github.com/sabeurthabti/react-native-css)(Web CSS转换器)
3. [react-native-style-tachyons](https://github.com/tachyons-css/react-native-style-tachyons)(预设样式布局)
4. [css-to-react-native](https://github.com/styled-components/css-to-react-native)(也是Web CSS转换器,易用度比第一个略差)
...
剩下大概都是些预处理器，终究没有能够有个完整解决方案。


### React-Native 样式管理解决方案探索

1. 自定义继承规则
2. 命名空间
3. 全局变量
4. 样式变量继承
5. 样式对象继承
6. 样式属性集合

伪代码的解决方案，大概是如此：

```
Styles = cssTree(GlobalStyle)(Middleware)(Style);
```

**GlobalStyle**为全局主题配置
**Middleware**为自定义继承的中间件
**Style**为组件内的样式表

**于是全文安利的项目就是：[react-native-css-tree](https://github.com/unadlib/react-native-css-tree)**
[https://github.com/unadlib/react-native-css-tree](https://github.com/unadlib/react-native-css-tree)

它解决了刚才提到的六点特性中前五个。使用 `cssTree(GlobalStyle)(Middleware)` 封装为一个公共的主题式模块，并且完整实现所有继承与命名空间。

使用例子：

```
import cssTree from 'react-native-css-tree';

export default class example extends Component {
    render() {
        return (
            <View style={styles.container}>
                <Text style={styles.container.welcome}></Text>
                <Text style={styles.container.instructions}></Text>
                <Text style={styles.container.instructions}></Text>
            </View>
        );
    }
}

const style = {
    container: {
        flex: 1,
        justifyContent: 'center',
        alignItems: 'center',
        backgroundColor: "$mainColor",
        welcome: {
            fontSize: "$fontSize",
            textAlign: '$alignItems',
            margin: "$grid",
        },
        instructions: {
            textAlign: '$alignItems',
            color: '$textColor',
        },
    },
};

const styles = cssTree({
    mainColor: '#00d1ff',
    otherColor: '#fff',
    textColor: "#333333",
    fontSize: 20,
    backgroundColor: "red",
    grid: 10,
})(function (key, parent, sub) {
    if(key==="welcome") sub.color = parent.otherColor;
    return sub
})(style);
```

如果感觉不错，欢迎去star一下。
**[react-native-css-tree](https://github.com/unadlib/react-native-css-tree)**
ps.后续将完善功能：margin/padding/border/shadow/radius...等相关样式集合属性。