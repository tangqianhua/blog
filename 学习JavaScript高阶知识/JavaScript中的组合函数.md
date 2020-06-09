# JavaScript 中的组合函数

这骗文章我们一起来学习下组合函数，首先我们要知道 `组合`  是什么意思？**组合其实就是将多个东西拼成一个新的东西**。比如我们现在的开发团队，就是由多个开发人员组成而成的。所以组合函数，就是将多个函数组成在一起。<br />

<a name="dmlZG"></a>

### pipe

`pipe`  的中文是 `管道`  的意思，它代表的是一条从左往右的管道。它结合了 N 个功能，调用内部的每个函数，最后再输出。<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/512535/1591627097548-c21f9f9c-f0f8-4848-9937-46bb710993f5.png#align=left&display=inline&height=134&margin=%5Bobject%20Object%5D&name=image.png&originHeight=268&originWidth=1088&size=29766&status=done&style=none&width=544)<br />比如我们现在要定义一个函数，返回一个人的名字：

```javascript
function getName(people) {
  return people.name;
}
getName({ name: "tqh" });
```

然后我们还想要一个函数，实现大写的功能：

```javascript
function toUpperCase(str) {
  return str.toUpperCase();
}
toUpperCase("tqh");
```

但是现在，我们既要返回一个人的名字，还要将这个人的名字全部大写，于是我们会这样调用函数：

```javascript
const name = toUpperCase(getName({ name: "tqh" }));
```

当然也有人会觉得，只要把两个方法合并下就行了，如下所示：

```javascript
function getName(people) {
  return people.name.toUpperCase();
}
```

如果有一天，我想让 getName 方法只返回 name，而不多做其他的处理的？最好不要让一个函数做太多的事情，不然维护起来很麻烦。<br />
<br />但是你有没有发现 `toUpperCase(getName({ name: "tqh" }))`  这种写法太臃肿了，一直嵌套在里面，如果我还要对返回值做截取，我们是不是要这样做：

```javascript
function cutOut(str) {
  return str.slice(0, 2);
}
cutOut(toUpperCase(getName({ name: "tqh" })));
```

当我们的功能越来越多，函数嵌套的就越来越深，写代码的人也难受，看代码的人更难受。<br />
<br />于是我们要优化一下它，导致嵌入较深的原因是因为调用的方法较多，每个方法都是要拿到上一个方法返回的结果再执行，如果我们可以将这些方法依次传入，然后再按照循序调用是不是就可以了，比如：<br />pipe(getName, toUpperCase, cutOut)<br />
<br />定义一个函数 `pipe` ，接受 N 个参数，按照顺序传入，然后再将这些参数进行合并操作：

```javascript
function pipe(...fns) {
  return function (metadata) {
    return fns.reduce((v, f) => f(v), metadata);
  };
}
```

我们可以看到，在 reduce 中，每个函数取上一个函数的输出，最后会将所有的函数进行处理，输出最后的一个值<br />
<br />我们可以打开谷歌的控制台，输出下面这段代码：

```javascript
function getName(people) {
  return people.name;
}

function toUpperCase(str) {
  return str.toUpperCase();
}

function cutOut(str) {
  return str.slice(0, 2);
}

function pipe(...fns) {
  return function (metadata) {
    return fns.reduce((v, f) => f(v), metadata);
  };
}

pipe(getName, toUpperCase, cutOut)({ name: "tqh" });
```

我们在第 15 行代码中，打了一个 debugger，我们来看看控制的效果：<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/512535/1591674073285-e43c1721-dce8-477b-ad32-cacdd4ec0da7.png#align=left&display=inline&height=273&margin=%5Bobject%20Object%5D&name=image.png&originHeight=328&originWidth=895&size=84099&status=done&style=none&width=746)<br />我们可以看到。当前函数执行的作用域中，metadata 是{name: 'tqh'}, 闭包中的 fns 是个数组，存放的是[getName, toUpperCase, cutOut]，这三个函数。<br />
<br />当代码第一次执行到第 16 行的时候，它会执行 reduce 方法：<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/512535/1591674311649-35b46397-262f-479f-95a4-72f9f21b263b.png#align=left&display=inline&height=372&margin=%5Bobject%20Object%5D&name=image.png&originHeight=440&originWidth=882&size=90151&status=done&style=none&width=746)<br />这个时候会执行 `getName函数` ，并且 `v`  是{name: "tqh"}, getName 执行完成后，会执行 toUpperCase 函数<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/512535/1591674592369-605e87d3-e5f3-46b3-ab69-86e9d348c9bc.png#align=left&display=inline&height=421&margin=%5Bobject%20Object%5D&name=image.png&originHeight=443&originWidth=763&size=86546&status=done&style=none&width=725)<br />这个时候 `v`  是“tqh”，最后执行 cutOut 函数，将最后的结果 return 出来，结果为： `"TQ"` 。<br />
<br />这就是 pipe 的好处，通过 reduce，合并多个函数，将函数依次处理并输出结果。<br />

<a name="5IshN"></a>

### compose

compose 跟 pipe 是反方向操作，如果要跟 pipe 的处理一样，就是反向操作，所以它的处理方式应该是这样的：<br />

```javascript
compose(cutOut, toUpperCase, getName)({ name: "tqh" });
```

如果要按照这些顺序是执行，执行结果又要一样，应该怎么取做呢？好好想一想，认真想一想。<br />

```javascript
function compose(...fns) {
  return function (metadata) {
    return fns.reduceRight((v, f) => f(v), metadata);
  };
}
```

是不是很简单，你也可以自己 debugger 试一试，这里多处理了，原理是一样的。<br />

<a name="Cg5EE"></a>

### 总结

多一个事情进行多次不同的操作，需要定义很多方法，最开始的嵌入调用比较恶心，看起来不美观。<br />
<br />通过 `pipe`  管道式的方式来处理原先的嵌入式掉用，通过 reduce 来封装一个方法。<br />
<br />compose 跟 pipe 一样，只不过它是反向操作，实现原理是一样的。
