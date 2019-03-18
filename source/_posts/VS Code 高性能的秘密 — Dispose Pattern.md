---
title: VS Code 高性能的秘密 — Dispose Pattern
date: 2019-03-18 19:23:40
thumbnail: https://haitao.nos.netease.com/022f0fd9-d1ee-4a2e-8372-5da19adf1238_225_224.png
tags: ['vscode', 'design pattern']
---

VS Code 是微软出品的新一代轻量级编辑器，一出道便以简洁大气的界面、卓越的性能、以及灵活的可扩展性吸引了大批的拥趸。

![VS Code Icon](https://haitao.nos.netease.com/2b129013-9a04-467f-8b46-b0ebe43b5fea_225_224.png)

插件化是 VS Code 的精髓，大部分功能比如 command、key binding、context menu 都是通过它对外提供的一套扩展[API](https://code.visualstudio.com/api)实现并集成到 Code 中。 <!-- more --> VS Code 使用多进程的架构来分别处理编辑器的渲染与执行，每开一个窗口，便会为该窗口创建一个进程执行插件，该进程即为 Extension Host。Extension Host 按需激活插件，同一时间内，插件代码可能被运行多次。

![VS Code Architectur](https://static001.geekbang.org/resource/image/d9/9b/d94bcbb1ff5e097660a9262cf485039b.png)

为了保证插件的高效运行，VS Code 使用了 Dispose 模式，大部分插件 API 都实现了 IDisposable 接口，生成的对象则会拥有一个 dispose 函数属性。

```js
interface IDisposable {
	dispose(): void;
}
```

Dispose 模式主要用来资源管理，资源比如内存被对象占用，则会通过调用方法来释放，这些方法通常被命名为‘close’，‘dispose’，‘free’，‘release’。一个著名的例子便是 C#，C#通过 Dipose Pattern 来释放不受 CLR(Common Language Runtime)管理的非托管资源。

VS Code 是由 Javascript 实现的，众所周知，Javascript 的内存分配是通过 GC(garbage collector)进行管理，大部分情况下它都是自动执行且对用户不可见的。然而这种自动化的管理方式却存在一个潜在的问题，就是 Javascript 开发者会错误的认为他们不需要再关心内存管理了，从而再无意间书写一些不利于内存回收的代码。

所以，最清楚被分配的内存在未来是否需要使用的还是开发者，但是每次使用完一个对象后就手动的将其销毁，这样的做法即不高效，也不可靠。正因为此，VS Code 使用了 Dispose Pattern 来管理对象销毁。当扩展功能执行时，Extension Host 会在正确的时机调用 dispose 方法，销毁 Code 生成的对象，减少内存使用。比如说，方法‘setStatusBarMessage(value: string)’返回一个‘Disposable’对象，当调用 dispose 方法的时候会移除掉信息对象。

![Dispose Pattern](https://haitao.nos.netease.com/94397bbc-f94b-4015-a7c2-ec53cf71e197_589_311.jpg)

Dispose pattern 的实现如下

```js
// 第一个重载参数是单个disposable类型
function dispose<T extends IDisposable>(disposable: T): T;
// 第二个重载参数是多个disposable类型传参数，参数可能为undefined。
function dispose<T extends IDisposable>(...disposables: Array<T | undefined>): T[];
// 第三个重载参数是一个disposable类型的数组。
function dispose<T extends IDisposable>(disposables: T[]): T[];
// 第三个重载参数为两种，第一个是disposable类型或disposable数组类型，剩余的为disposable类型。
function dispose<T extends IDisposable>(first: T | T[], ...rest: T[]): T | T[] | undefined {
  // 如果第一个参数是数组，则依次调用传参数的dispose方法
  if (Array.isArray(first)) {
  	first.forEach(d => d && d.dispose());
  	// 返回空的数组
  	return [];
  } else if (rest.length === 0) {
   // 如果没有没有剩余参数
  	if (first) {
  	   // 如果存在first
  	   // 调用第一个dispose
  		first.dispose();
  	  // 返回first
  		return first;
  	}

  	return undefined;
  } else {
    // first不是数组，且rest长度不为0
  	dispose(first);
  	dispose(rest);

  	// 返回空数组
  	return [];
  }
}

// implement IDisposable 的Disposable 抽象类
abstract class Disposable implements IDisposable {

  // Disposable类的静态对象，用于返回一个包含空的dispose方法的IDisposable对象。dispose被执行了，则表示该对象不再需要了。
  // 部分基础API使用了该对象，用于标志资源释放。
	static None = Object.freeze<IDisposable>({ dispose() { } });

  // protected属性toDispose返回protected对象_toDispose, 该对象初始值是一个空的数组。
	protected _toDispose: IDisposable[] = [];
	// 返回IDisposable数组。
	protected get toDispose(): IDisposable[] { return this._toDispose; }

  // 设置状态标志，表示该对象是否有被销毁。
	private _lifecycle_disposable_isDisposed = false;

  // 暴露公共方法dispose，执行完后将_lifecycle_disposable_isDisposed状态标志设为true，同时调用lifecycle内的dispose方法处理_toDispose数组，并重新赋值空数组。
	public dispose(): void {
		this._lifecycle_disposable_isDisposed = true;
		this._toDispose = dispose(this._toDispose);
	}

  // 内部方法注册实例，若_lifecycle_disposable_isDisposed为true，则表明该方法已经被dispose过，则不能再使用，需dispose掉，否则，推入_toDispose数组。
	protected _register<T extends IDisposable>(t: T): T {
	  // 判断这个对象有没有被dispose过
		if (this._lifecycle_disposable_isDisposed) {
			console.warn('Registering disposable on object that has already been disposed.');
			t.dispose();
		} else {
			this._toDispose.push(t);
		}

		return t;
	}
}
```

扩展 API 大部分功能类或功能方法都通过上面的抽象类 Disposable 或接口 IDisposable 实现 dispose 方法。下面的函数示例了一个功能类 DelayedDragHandler 如何实现 dispose 方法，当 HTMLElement 的延迟拖动方法执行完后，其实例对象的 timeout 对象会被及时清除，避免内存占用。

![dispose object](https://haitao.nos.netease.com/0a050e4d-f7f2-417d-9505-98c33bda36df_401_416.jpg)

```js
/**
 * A helper that will execute a provided function when the provided HTMLElement receives
 *  dragover event for 800ms. If the drag is aborted before, the callback will not be triggered.
 */
export class DelayedDragHandler extends Disposable {
	private timeout: any;

	constructor(container: HTMLElement, callback: () => void) {
		super();

		this._register(addDisposableListener(container, 'dragover', () => {
			if (!this.timeout) {
				this.timeout = setTimeout(() => {
					callback();

					this.timeout = null;
				}, 800);
			}
		}));
	}

	private clearDragTimeout(): void {
		if (this.timeout) {
			clearTimeout(this.timeout);
			this.timeout = null;
		}
	}

	dispose(): void {
		super.dispose();

		this.clearDragTimeout();
	}
}
```

引用：

-   [Dispose Pattern](https://en.wikipedia.org/wiki/Dispose_pattern)
-   [实现标准的 Dispose 模式](https://wizardforcel.gitbooks.io/effective-csharp/content/19.html)
-   [Memory Management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management#Mark-and-sweep_algorithm)
-   [Garbage collection](https://javascript.info/garbage-collection)
-   [Garbage collection in V8, an illustrated guide](https://github.com/lrlna/sketchin/blob/master/guides/garbage-collection-in-v8.md#-sourcesjs)
-   [API](https://code.visualstudio.com/api)
