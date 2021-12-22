Slate

## 输入简单文字流程 (macos 11.4, Chrome 96.0.4664.110)

输入 a，浏览器依次触发：

- keydown
  - KeyboardEvent {isTrusted: true, key: 'a', code: 'KeyA', location: 0, ctrlKey: false, …}
  - 不是快捷键操作，如 ctrl-b，不做任何事情
- beforeinput (real)
  - InputEvent {isTrusted: true, data: 'a', isComposing: false, inputType: 'insertText', dataTransfer: null, …}
  - 真正的 DOM 事件
  - 设置 deferredOperations -> Editor.insertText() 准备更新 editor 维护的状态
- beforeinput (react)
  - TextEvent {isTrusted: true, data: 'a', view: Window, detail: 0, sourceCapabilities: null, …}
  - React 封装的事件，不是真事件
  - 对于正常浏览器(支持真.beforeinput 事件的），不做任何事，否则 Editor.insertText(editor, data)
- (DOM) 此时 dom 有 'a'
- input
  - InputEvent {isTrusted: true, data: 'a', isComposing: false, inputType: 'insertText', dataTransfer: null, …} [ƒ]
  - 执行 deferredOperations （上面的 Editor.insertText(editor, data))，触发 react 重绘

## 用输入法（IME）输入文字流程 (macos 11.4, Chrome 96.0.4664.110, 搜狗)

输入拼音串 w<space> 来输入「我」，浏览器依次触发：

- keydown
  - KeyboardEvent {isTrusted: true, key: 'w', code: 'KeyW', location: 0, ctrlKey: false, …}
  - 不是快捷键操作，如 ctrl-b，不做任何事情
- compositionstart
  - CompositionEvent {isTrusted: true, data: '', view: Window, detail: 0, sourceCapabilities: InputDeviceCapabilities, …}
  - 如果选中东西，则删除
- beforeinput (real)
  - InputEvent {isTrusted: true, data: 'w', isComposing: true, inputType: 'insertCompositionText', dataTransfer: null, …}
  - e.type 是 insertCompositionText，不作任何事情
- compositionupdate
  - CompositionEvent {isTrusted: true, data: 'w', view: Window, detail: 0, sourceCapabilities: InputDeviceCapabilities, …}
  - 标记 isComposing=true，没有这个状态才处理 selection 变化/onKeyDown 事件
- input
  - InputEvent {isTrusted: true, data: 'w', isComposing: true, inputType: 'insertCompositionText', dataTransfer: null, …} []
  - 执行 deferredOperations （空）
- keydown
  - KeyboardEvent {isTrusted: true, key: ' ', code: 'Space', location: 0, ctrlKey: false, …}
  - isComposing===true，不做任何事情
- beforeinput (real)
  - InputEvent {isTrusted: true, data: '我', isComposing: true, inputType: 'insertCompositionText', dataTransfer: null, …}
  - e.type 是 insertCompositionText，不作任何事情（应该是为了避免触发 setState，输入被打断）
- compositionupdate
  - CompositionEvent {isTrusted: true, data: '我', view: Window, detail: 0, sourceCapabilities: InputDeviceCapabilities, …}
  - 标记 isComposing=true，没有这个状态才处理 selection 变化/onKeyDown 事件
- beforeinput (react)
  - TextEvent {isTrusted: true, data: '我', view: Window, detail: 0, sourceCapabilities: null, …}
  - React 封装的事件，不是真事件
  - 对于正常浏览器(支持真.beforeinput 事件的），不做任何事，否则 Editor.insertText(editor, data)
- input
  - InputEvent {isTrusted: true, data: '我', isComposing: true, inputType: 'insertCompositionText', dataTransfer: null, …} []
  - 执行 deferredOperations （空）
- compositionend
  - CompositionEvent {isTrusted: false, data: '我', view: Window, detail: 0, sourceCapabilities: InputDeviceCapabilities, …}
  - 标记 isComposing=false
  - 如果是 Chrome， Editor.insertText(editor, event.data)
    - 为什么 Chrome 需要在这时插入？
    - 原因是 Chrome 不会发以下两个事件（safari 会发）
      - deleteCompositionText -- 忽略（DOM 变化：临时拼音串被删除），Chrome 也会改 DOM，但不发这个事件
      - insertFromComposition -- 此事件 native===false, 会 preventDefault 并 Editor.insertText() <-- Chrome 缺少这个，因此需要补上

## Editor.insertText 做了什么

- 调用 Transform.insertText, Editor.apply 更新 json 状态
- 延迟调用 onChange（React.unstable_batchedUpdates），外部如果正确 setState, 则可以触发 Slate 组件重绘
- <Slate> 的 Children 负责将 json 状态映射为 React 节点树，由 React 完成 dom 修改
  - 同时借由 useLayoutEffect 维护 KEY_TO_ELEMENT, NODE_TO_ELEMENT, ELEMENT_TO_NODE 等映射关系
  - 为了让 react 能正确调和 dom，必须让 json/dom 保持准确的对应关系？
