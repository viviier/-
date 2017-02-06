[译]驯服ES7的异步野兽 - async

> * 原文链接： [Taming the asynchronous beast with ES7](https://pouchdb.com/2015/03/05/taming-the-async-beast-with-es7.html)
> * 作者： [Nolan Lawson](https://twitter.com/nolanlawson)
> * 译者： [ant](http://www.viviier.com)

在PouchDB里面最棘手的一个问题是它的异步API。我在Stack Overflow，Github和IRC上看见过很多关于此类的问题和困惑，并且关于callbacks和promises几乎也有很多人经常误解。

PouchDB是对IndexedDB，WebSQL，LevelDB和CouchDB的一个抽象化过程。PouchDB所有的API都是异步的，因此PouchDB也必须是异步的。

有时我们想要去优雅的使用database的API，然而在真正使用的时候我们仍然会受到‘从简’的思维影响去使用相对简单的LocalStorage：

```javascript
if(!localStorage.foo) {
	localStorage.foo = 'bar';
};
console.log(localStorage.foo);
```

使用LocalStorage时，你只需要简单的将它视为碰巧可以保存你的数据的一个神奇的JavaScript对象，它使用同步的方法来处理JavaScript本身。而对于LocalStorage存在的一些问题，它人性化的API说明也使得它能够继续流行下去。人们继续使用LocalStorage，是因为它简单并且它的工作思维在用户的预期之中（同步）。

### Promises 不是万能药

对于PouchDB，我们尝试使用异步的API promise来减轻它的复杂度，借此来帮助我们摆脱恐怖的“回调地狱”。

然而promise代码仍然很生硬，因为promise只是替换了原来的写法，如<code>try</code>，<code>catch</code>和<code>return</code>：

```javascript
var db = new PouchDB('mydb);
db.porst({}).then(function (result){
	return db.get(result.id);
}).then(function (doc){
  	console.log(doc);
}).catch(function (err){
  	console.log(err);
});
```

作为JavaScript开发者，我们现在的脑海中需要有一条直线，上面有着一个并行的系统 - 同步和异步。随着我们的控制流越来越多越来越复杂，这种方式就会变的比较糟糕。为此，我们需要去学习像<code>Promise.all()</code>和<code>Promise.resolve()</code>这样的API，或者我们只是在众多的javascript库中找到了一个我们能够读懂并且能够帮助我们解决这个问题的库。

直到最近，这都是我们最好到的解决方法，但是ES7的出现改变了这一切。

### 拥抱ES7

当你知道了什么是ES7之后，你可以像这样重写上面的代码：

```javascript
let db = new PouchDB('mydb')
try {
	let result = await db.post({})
	let doc = await db.get(result.id)
 	console.log(doc)
} catch (err){
  	console.log(err)
}
```

并且，我们能够这么舒服的去书写这种代码也要感谢像Babel.js，Regenerator这样能把ES6/7代码转换为ES5运行浏览器上的工具。


### Async 函数
ES7给了我们一个新的函数类，<code>async function</code>，在async函数内，我们有一个新的关键字，<code>await</code>，我们可以用它来等待一个promise函数。

```javascript
async function myFunction() {
	let result = await somethingThatReturnsAPromise()
	console.log(result)
}
```

如果promise返回状态为resolve，我们就可以立即进行下一条代码，如果返回状态为reject，那么会抛出一条错误，所以<code>try/catch</code>实际上也在工作。

```javascript
async function myFunction() {
	try {
		await somethingThatReturnsAPromise()
	} catch (err) {
		console.log(err)
	}
}
```

async让我们写出来的代码看起来像是同步，但实际上是异步的。还有一个细节是事实上这个API返回的是一个promise而没有去阻塞事件的进行。

![pic](https://pouchdb.com/static/img/pepperidge_farm_remembers.png)

还有一个async的特性是，我们能能够使用任何能够返回promise函数的库。PouchDB就是这样的库，所以让我们去测试一下吧。

### 处理错误以及返回值

首先，让我们考虑一个在PouchDB里常见的一个用法：如果_id存在的话，我们想通过<code>_id</code>来<code>get()</code>一个文件，否则我们返回一个新的文件.

使用promise，你应该会这样写：

```javascript
db.get('docid).catch(function (err) {
	if(err.name === 'not_found') {
		return {};  // new doc
	}
	throw err;  // 404
}).then(function (doc) {
	console.log(doc);
})
```

如果使用async函数，就可以这样来写：

```javascript
let doc
try {
	doc = await db.get('docid')
} catch (err) {
	if(err.name === 'not_found') {
		doc = {}
	} else {
		throw err
	}
}
console.log(doc)
```

可读性上升！这是几乎相同的代码，<code>if db.get()</code>直到返回一个新的doc而不是一个promise。它们之间唯一的不同是我们添加了一个<code>await</code>关键字来处理我们所调用的任何的返回pormise的函数。

### 潜在陷阱
还有一些很细微的问题，我很高兴我能在使用的过程中遇到它们。

首先，无论你在时候什么地方使用<code>await</code>，你都需要把它放在async函数内，所以如果你的代码严重依赖PouchDB，你可能发现你会写很多关于异步（async）的函数，而相对写的普通的函数就很少。

另一方面，更加需要注意的一个问题是你的代码需要小心翼翼的放在<code>try/catch</code>中，否则当promise返回reject的时候，错误不会被抛出，因此你看不到代码的问题所在。

我的忠告是确保你的异步代码被<code>try/catch</code>完全包围住。

```javascript
async function createNewDoc() {
	let response = await db.post({})
	return await db.get(response.id)
}

async function printDoc() {
	try {
		let doc = await createNewDoc()
		console.log(doc)
	} catch (err) {
		console.log(err)
	}
}
```

### 循环
当使用异步函数去处理迭代时令人印象深刻。举个例子，我们想要插入一些文档到数据库里面，但是对于执行顺序，我们想要promise函数执行一个之后再执行另外一个，而不是同时执行。

遵循ES6规范的promise，我们可以去链式调用我们的promise函数：

```javascript
var promise = Promise.resolve();
var docs = [{}, {}, {}];

docs.forEach(function (doc) {
	promise = promise.then(function () {
		return db.post(doc)
	});
});

promise.then(function () {
	// 现在我们所有的docs都被保存
});

```

这种代码不够优雅，并且也容易引发错误，因为如果你偶然的这么写了：

```javascript
docs.forEach(function (doc) {
	promise = promise.then(db.post(doc));
})

```

然后这个promise其实将会并发执行，这样会发生意想不到的结果。


使用ES7，虽然我们可以只使用一个简单的for循环：

```javascript
let docs = [{}, {}, {}]

for(let i = 0; i < docs.length; i++) {
	let doc = docs[i]
	await db.post(doc)
}

这个非常简洁的代码和链式调用promise相同。我们甚至能够使它更简洁，通过使用<code>for...of</code>：

```javascript
let docs = [{}, {}, {}]

for(let doc of docs) {
	await db.post(doc)
}
```

注意这里不能使用<code>forEach()</code>循环，如果你这么写了：

```javascript
let docs = [{}, {}, {}]

// WARNING: this won't work
docs.forEach(function (doc) {
	await db.post(doc)
})
```

然后Babel.js将会报出一个错误：

```javascript
Error : /../script.js: Unexpected token (38:23)
> 38 |     await db.post(doc);
     |           ^
```

这是因为你不能使用<code>await</code>在普通函数中。你需要一个async函数。

然而，如果你尝试去使用async函数，那么你将会遇到一些奇怪的bug：

```javascript
let docs = [{}, {}, {}]

// WARNING: this won't work
docs.forEach(async function (doc, i) {
	await db.post(doc)
	console.log(i)
})
console.log('main loop done')
```

这段代码可以编译，但在输出的时候就会看到问题：

```javascript
main loop done
0
1
2
```

发生了什么让主函数提前退出，是因为<code>await</code>其实是在子函数里，此外，它将会同时执行每一个promise，这并不是我们想要的结果。

所以这告诉我们，当你的异步函数中有任何的函数时一定要小心。<code>await</code>只会暂停它的父函数，所以在检查的时候要注意，它在做的，以及实际上你要他做的，是不是同一件事。

### 并发循环
如果我们想要同时执行很多的promise，那么使用ES7就能简单优雅的完成。

回到ES6的promise，我们可以使用<code>Promise.all()</code>。使用它从promise数组里来返回一个数组值：

```javascript
var docs = [{}, {}, {}];

return Promise.all(docs.map(function (doc) {
	return db.post(doc);
})).then(function (results) {
	console.log(results);
});
```

在ES7，我们可以使用更直观的方法：

```javascript
let docs = [{}, {}, {}]
let promises = docs.map((doc) => db.post(doc))

let results = []
for(let promise of promises) {
	result.push(await promise)
}
console.log(results)
```

这段代码重要的结构是 1) 创建<code>promises</code>数组，然后立即开始调用所有的promise函数 2)然后我们在主函数中使用<code>await</code>来处理promise。如果我们尝试了使用<code>Array.prototype.map</codde>，那么它将不会工作：

```javascript
let docs = [{}, {}, {}]
let promises  = docs.map((doc) => db.post(doc))

// WARNING: this doesn't work
let results = promises.map(async function(promise) {
	return await promise
})

// This will just be a list of promise :(
console.log(results)
```

这段代码不会工作的原因是我们使用<code>await</code>时把它放在了子函数内，而不是主函数，所以主函数退出之前我们就完成了waiting。

如果你不介意使用<code>Promise.all</code>，你也可以使用它来整理你的代码：

```javascript
let docs = [{}, {}, {}]
let promises = docs.map((doc) => db.post(doc))

let results = await Promise.all(promises)
console.log(results)
```

如果我们能够使用数组推导（array comprehesions）想必看起来可以更好。然而，现在它还不算太规范，所以目前还不被Regenerator支持。

### 注意事项
ES7仍然很流行。但Node.js和io.js还不支持Async函数，你需要去加载一些实验性的插件才能让Babel编译它。就目前来说，async/await规范仍然处于“提案”状态。

对此，你需要一些ES6的代码转换工具包括Regenerator和ES6的编译工具使得代码能够工作在ES5的浏览器上。对于我来说，这些工具加起来差不多60KB，代码压缩和gzipped。然而对于大多数开发者来说，这些工具太多，水很深。

但是这些新的工具使用起来非常有趣，使用它们时工作就像是用颜料涂满画卷，而且关于异步的库也让我仿佛看到了ES7美好的未来。

所以，如果你也想玩一下，你可以点击进入我写的一个小的[demo库](https://github.com/nolanlawson/async-functions-in-pouchdb)。开始运行它，只需要解压代码，然后运行<code>npm install && npm run build</code>。你也可以去了解更多关于ES7的内容，或是去观看Jafar Husain在youtube上的[演讲](https://www.youtube.com/watch?v=DqMFX91ToLw)。

### 结论
异步函数在ES7里是一个新的概念，它们让之前不在异步中使用的<code>return</code>和<code>try/catch</code>回归，并且鼓励我们用同步的代码来书写异步，虽然看起来像老旧的代码，但是它拥有着更高的性能。

更重药的是，异步函数使得像PouchDB这样的API更容易工作。所以希望这将能改变少量用户错误和混乱的代码，并且让我们能写出更具优雅性和可读性的代码。

而且谁也不知道，也许人们最后将会放弃LocalStorage，并选择相对来说更现代的客户端数据库。