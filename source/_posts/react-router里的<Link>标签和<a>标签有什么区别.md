---
title: react-router里的&#60;Link&#62;标签和&#60;a&#62;标签有什么区别?
catalogs: 面试题
---

先看 Link 点击事件 handleClick 部分源码

```javascript
if (_this.props.onClick) _this.props.onClick(event);

if (
  !event.defaultPrevented && // onClick prevented default
  event.button === 0 && // ignore everything but left clicks
  !_this.props.target && // let browser handle "target=_blank" etc.
  !isModifiedEvent(event) // ignore clicks with modifier keys
) {
  event.preventDefault();

  var history = _this.context.router.history;
  var _this$props = _this.props,
    replace = _this$props.replace,
    to = _this$props.to;

  if (replace) {
    history.replace(to);
  } else {
    history.push(to);
  }
}
```

Link 做了 3 件事情：

1. 有 onclick 那就执行 onclick
2. click 的时候阻止 a 标签默认事件（这样子点击<a href="/abc">123</a>就不会跳转和刷新页面）
3. 再取得跳转 href（即是 to），如果没有传 replace 就默认使用 push，用 history（前端路由两种方式之一，history & hash）跳转，此时只是链接变了，并没有刷新页面

a 标签则跳转新页面，或者从另外一个 tab 打开新页面（锚点除外）。
