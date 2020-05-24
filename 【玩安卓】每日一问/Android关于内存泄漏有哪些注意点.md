
# Android关于内存泄漏有哪些注意点？

[原始网页直通车](https://www.wanandroid.com/wenda/show/8206)

> 内存泄漏的根本原因是一个长生命周期对象持有一个短生命周期对象，造成短生命周期对象没有办法被回收。

## 常见情形：

1. 内部类形式使用 `Handler` 并发送了延迟消息时，当退出`Activity` 时会造成 `Activity` 内存泄漏。 `AsyncTask` 同理。

    解决方案：
    
    1. `Activity` 销毁时移除所有与 `当前Handler` 相关联的消息。

    2. 将使用的 `Handler内部类` 定义成静态内部类。但是静态内部类引用不到外部类的非静态属性和方法，可以在内部类中使用弱引用持有外部类，通过弱引用去调用外部类的属性和方法，同时由于弱引用具有 *一个对象只被弱引用关联时则它在GC时会被直接回收* 的特性，所以不用担心有内存泄漏的风险。

2. 单例类持有 `Activity` 、 `Fragment` 、 `View` 等有生命周期的 `Context` 属性造成内存泄漏。

3. 各种注册操作没有反注册（广播、EventBus等）

4.  `Cursor` 、 `Input` 、 `OutputStream` 之类的用完后一定要及时关闭。操作这些对象， `close` 方法尽量放在 `finally` 块里，防止出异常时不能执行。

5. 布局引用 `WebView` 的场景。

    当 `Activity` 被关闭时， `WebView` 不会马上被 GC 回收，而是提交给事务，进行队列处理，这样就造成了内存泄漏，导致 `WebView` 无法及时回收。

    可行的解决方案：

    采用动态添加 `WebView` 的方式，并在页面 `onDestroy()` 时，先移除 `WebView` 的所有回调，并清除所有 `JavaScriptInterface` 操作，然后通过父布局移除 `WebView` 。

    示例代码：

    ```kotlin
    override fun onDestroy() {
        webView?.apply {
            val parent = parent
            if (parent is ViewGroup) {
                parent.removeView(this)
            }
            stopLoading()
            settings.javaScriptEnabled = false
            clearHistory()
            removeAllViews()
            destroy()
        }
    }
    ```

6. `Map` 和 `List` 中持有大量的数据，使用完没有及时的清理，也会造成内存泄漏。