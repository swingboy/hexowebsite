---
title: React v16.4.0 事件原理
date: 2018-07-26 18:54:20
tags:
---



（针对v16.4.0）
### 原生事件系统
我们通常监听真实DOM。举🌰来说，我们想监听按钮的点击事件，那么我们在按钮DOM上绑定事件和对应的回调函数即可。

>遗憾的是若页面复杂且事件处理频率高，那么对网页性能是个考验。

### React事件系统
react的事件处理再眼花缭乱终究还是要回归原生的事件系统。

### react事件绑定
react 中的事件是根据 nativeEvent 对象修改后的合成对象，对于事件采用事件委托，将事件绑定在顶层 document 对象上。对于全局 dom 元素中所涉及的事件， react 都会处理。对于冒泡事件，是在 document 对象的冒泡阶段触发。对于非冒泡事件，是在 document 对象的捕获阶段触发。最后在 dispatchEvent 中决定真正回调函数的执行。

### 合成事件
事件处理程序通过 合成事件（SyntheticEvent）的实例传递，SyntheticEvent 是浏览器原生事件跨浏览器的封装。SyntheticEvent 和浏览器原生事件一样有 stopPropagation()、preventDefault() 接口，而且这些接口夸浏览器兼容。

如果出于某些原因想使用浏览器原生事件，可以使用 nativeEvent 属性获取。每个和成事件（SyntheticEvent）对象都有以下属性：

boolean bubbles
boolean cancelable
DOMEventTarget currentTarget
boolean defaultPrevented
Number eventPhase
boolean isTrusted
DOMEvent nativeEvent
void preventDefault()
void stopPropagation()
DOMEventTarget target
Date timeStamp
String type

React实现了SyntheticEvent层处理事件

详细来说，React并不像原生事件一样将事件和DOM一一对应，而是将所有的事件都绑定在网页的document，通过统一的事件监听器处理并分发，找到对应的回调函数并执行。按照官方文档的说法，事件处理程序将传递SyntheticEvent的实例。



### react提出统一事件处理的目的主要是：
1.浏览器兼容性
2.改善性能
在复杂的UI系统中，越多的事件处理意味着应用需要消耗的内存越多。手动解决这个问题并不是很麻烦，但是在冗长的过程中，你需要尝试去给在来源于同一个父节点的事件进行分组。很难权衡。而React就帮我们解决了这个问题。
React不是直接的将事件挂在dom上，它在document的根部用一个事件处理去监听和调用相应的事件。


```
/*
 * Overview of React and the event system:
 *
 * +------------+    .
 * |    DOM     |    .
 * +------------+    .
 *       |           .
 *       v           .
 * +------------+    .
 * | ReactEvent |    .
 * |  Listener  |    .
 * +------------+    .                         +-----------+
 *       |           .               +--------+|SimpleEvent|
 *       |           .               |         |Plugin     |
 * +-----|------+    .               v         +-----------+
 * |     |      |    .    +--------------+                    +------------+
 * |     +-----------.--->|EventPluginHub|                    |    Event   |
 * |            |    .    |              |     +-----------+  | Propagators|
 * | ReactEvent |    .    |              |     |TapEvent   |  |------------|
 * |  Emitter   |    .    |              |<---+|Plugin     |  |other plugin|
 * |            |    .    |              |     +-----------+  |  utilities |
 * |     +-----------.--->|              |                    +------------+
 * |     |      |    .    +--------------+
 * +-----|------+    .                ^        +-----------+
 *       |           .                |        |Enter/Leave|
 *       +           .                +-------+|Plugin     |
 * +-------------+   .                         +-----------+
 * | application |   .
 * |-------------|   .
 * |             |   .
 * |             |   .
 * +-------------+   .
 *                   .
 *    React Core     .  General Purpose Event Plugin System
 */
```


#### SyntheticEvent 

```
/*
 * @param {object} dispatchConfig Configuration used to dispatch this event.
 * @param {*} targetInst Marker identifying the event target.
 * @param {object} nativeEvent Native browser event.
 * @param {DOMEventTarget} nativeEventTarget Target node.
* /
function SyntheticEvent(dispatchConfig, targetInst, nativeEvent, nativeEventTarget) {

  this.dispatchConfig = dispatchConfig;
  this._targetInst = targetInst;
  this.nativeEvent = nativeEvent;

  var Interface = this.constructor.Interface;
  for (var propName in Interface) {
    if (!Interface.hasOwnProperty(propName)) {
      continue;
    }
    var normalize = Interface[propName];
    if (normalize) {
      this[propName] = normalize(nativeEvent);
    } else {
      if (propName === 'target') {
        this.target = nativeEventTarget;
      } else {
        this[propName] = nativeEvent[propName];
      }
    }
  }

  var defaultPrevented = nativeEvent.defaultPrevented != null ? nativeEvent.defaultPrevented : nativeEvent.returnValue === false;
  if (defaultPrevented) {
    this.isDefaultPrevented = emptyFunction.thatReturnsTrue;
  } else {
    this.isDefaultPrevented = emptyFunction.thatReturnsFalse;
  }
  this.isPropagationStopped = emptyFunction.thatReturnsFalse;
  return this;
}

_assign(SyntheticEvent.prototype, {
  preventDefault: function () {
    this.defaultPrevented = true;
    var event = this.nativeEvent;
    if (!event) {
      return;
    }

    if (event.preventDefault) {
      event.preventDefault();
    } else if (typeof event.returnValue !== 'unknown') {
      event.returnValue = false;
    }
    this.isDefaultPrevented = emptyFunction.thatReturnsTrue;
  },

  stopPropagation: function () {
    var event = this.nativeEvent;
    if (!event) {
      return;
    }

    if (event.stopPropagation) {
      event.stopPropagation();
    } else if (typeof event.cancelBubble !== 'unknown') {
      event.cancelBubble = true;
    }

    this.isPropagationStopped = emptyFunction.thatReturnsTrue;
  },

  destructor: function () {
    var Interface = this.constructor.Interface;
    for (var propName in Interface) {
      {
        Object.defineProperty(this, propName, getPooledWarningPropertyDefinition(propName, Interface[propName]));
      }
    }
    for (var i = 0; i < shouldBeReleasedProperties.length; i++) {
      this[shouldBeReleasedProperties[i]] = null;
    }
    ...
  }
});

SyntheticEvent.Interface = EventInterface;

新建事件对象时，会重写stopPropagation（阻止冒泡） 和 preventDefault（阻止默认行为）这样的原生事件。


//我们所熟知事件接口 ，nativeEvent对象属性
var EventInterface = {
  type: null,
  target: null,
  currentTarget: emptyFunction.thatReturnsNull,
  eventPhase: null,
  bubbles: null,
  cancelable: null,
  timeStamp: function (event) {
    return event.timeStamp || Date.now();
  },
  defaultPrevented: null,
  isTrusted: null
};
```

还有下面的这些合成对象，这些对于事件类别不同，而有不同的属性
MouseEvent,AnimationEvent, UIEvent, InputEvent, CompositionEvent, ClipboardEvent, FocusEvent, KeyboardEvent, DragEvent, TouchEvent, TransitionEvent, WheelEvent上述的EventInterface基础上再添加




### 简单看下过程

![](.绑定/imgs/.png)

![](./imgs/触发.png)

#### 事件注册
事件的绑定在completeWork阶段

```
function completeWork(current, workInProgress, renderExpirationTime) {
    // ...
    var _instance = createInstance(type, newProps, rootContainerInstance, _currentHostContext, workInProgress);


    //接着
    if (finalizeInitialChildren(_instance, type, newProps, rootContainerInstance, _currentHostContext)) {
              markUpdate(workInProgress);
            }
    
    //.....
}


function createInstance(type, props, rootContainerInstance, hostContext, internalInstanceHandle) {
  var parentNamespace = void 0;
  {
    var hostContextDev = hostContext;
    validateDOMNesting$1(type, null, hostContextDev.ancestorInfo);
    if (typeof props.children === 'string' || typeof props.children === 'number') {
      var string = '' + props.children;
      var ownAncestorInfo = updatedAncestorInfo(hostContextDev.ancestorInfo, type, null);
      validateDOMNesting$1(null, string, ownAncestorInfo);
    }
    parentNamespace = hostContextDev.namespace;
  }
  debugger;
  var domElement = createElement(type, props, rootContainerInstance, parentNamespace);
  precacheFiberNode$1(internalInstanceHandle, domElement);
  updateFiberProps$1(domElement, props);
  return domElement;
}
/*
domElement等于真实创建的dom，里面调用的就是我们熟悉的createElement。而precacheFiberNode和updateFiberProps两个方法分别给domElement(真实dom)添加了两个属性。所有react16项目中的dom都会拥有这两个属性，并且这个两个属性的属性名在同一个项目中是一致的。
*/

//修改属性
function getFiberCurrentPropsFromNode$1(node) {
  return node[internalEventHandlersKey] || null;
}

//sck 更新属性 
//这里的props就是 所绑定元素的属性props
function updateFiberProps(node, props) {
  // debugger;
  node[internalEventHandlersKey] = props;
}

```

然后呢，就是监听document或者fragment上的事件

```
function ensureListeningTo(rootContainerElement, registrationName) {
  var isDocumentOrFragment = rootContainerElement.nodeType === DOCUMENT_NODE || rootContainerElement.nodeType === DOCUMENT_FRAGMENT_NODE;
  //就是document
  var doc = isDocumentOrFragment ? rootContainerElement : rootContainerElement.ownerDocument;
  listenTo(registrationName, doc);
}
```

在ensureListeningTo里调用listenTo方法

```
function listenTo(registrationName, mountAt) {
  debugger;
  var isListening = getListeningForDocument(mountAt);
  var dependencies = registrationNameDependencies[registrationName];

  for (var i = 0; i < dependencies.length; i++) {
    var dependency = dependencies[i];
    if (!(isListening.hasOwnProperty(dependency) && isListening[dependency])) {
      switch (dependency) {
        case TOP_SCROLL:
          trapCapturedEvent(TOP_SCROLL, mountAt);
          break;
        case TOP_FOCUS:
        case TOP_BLUR:
          trapCapturedEvent(TOP_FOCUS, mountAt);
          trapCapturedEvent(TOP_BLUR, mountAt);
          // We set the flag for a single dependency later in this function,
          // but this ensures we mark both as attached rather than just one.
          isListening[TOP_BLUR] = true;
          isListening[TOP_FOCUS] = true;
          break;
        case TOP_CANCEL:
        case TOP_CLOSE:
          if (isEventSupported(getRawEventName(dependency), true)) {
            trapCapturedEvent(dependency, mountAt);
          }
          break;
        case TOP_INVALID:
        case TOP_SUBMIT:
        case TOP_RESET:
          break;
        default:
          var isMediaEvent = mediaEventTypes.indexOf(dependency) !== -1;
          if (!isMediaEvent) {
            trapBubbledEvent(dependency, mountAt);
          }
          break;
      }
      isListening[dependency] = true;
    }
  }
}

在listenTo中调trapBubbledEvent。 其实埋点的监听的过程。


function trapBubbledEvent(topLevelType, element) {
  if (!element) {
    return null;
  }
  var dispatch = isInteractiveTopLevelEventType(topLevelType) ? dispatchInteractiveEvent : dispatchEvent;

  // 此时的listenner还是原来的
  addEventBubbleListener(element, getRawEventName(topLevelType),
  // Check if interactive and wrap in interactiveUpdates
  dispatch.bind(null, topLevelType));
  
}

function addEventBubbleListener(element, eventType, listener) {
  debugger;
  element.addEventListener(eventType, listener, false);
}
```

总结：completeWork->createInstance(更新属性等等)->setInitialDOMProperties->ensureListeningTo
->listenTo->trapBubbledEvent->addEventBubbleListener（进行我们熟悉的addEventListener）-> commitAllHostEffects 最后渲染完成


#### 二 事件合成
有点重合，逻辑有点重合。其实是前后脚的关系。

事件合成的过程，首先根据触发事件的target得到inst，然后遍历他的所有父节点,存储在局部遍历path中，记住这个path是有顺序关系的（后面可以解释react事件是如何阻止冒泡的）。得到path后进行遍历，假设遍历的组件同样注册了对应事件的listener，那么就挂载到event的_dispatchListeners和_dispatchInstances中去,这两个属性至关重要，后续的事件派发就是根据这两个属性进行的。 注意只有注册了对应事件的listener，才会挂载到event里面去。比如刚刚我们的ABC都绑定了Click，自然都会push到_dispatchListeners中去。


#### 三 事件派发

```
//callback is this func 注册事件的时候是listen的这个方法 
function dispatchInteractiveEvent(topLevelType, nativeEvent) {
  interactiveUpdates(dispatchEvent, topLevelType, nativeEvent);
}
```

```
function runExtractedEventsInBatch(topLevelType, targetInst, nativeEvent, nativeEventTarget) {
  debugger;
  //获得所有绑定的事件 看下面代码
  var events = extractEvents(topLevelType, targetInst, nativeEvent, nativeEventTarget);
  runEventsInBatch(events, false);
}

function runEventsInBatch(events, simulated) {
  alert('runEventsInBatch');
  if (events !== null) {
    eventQueue = accumulateInto(eventQueue, events);
  }
  .....
}

function extractEvents(topLevelType, targetInst, nativeEvent, nativeEventTarget) {
  var events = null;
  for (var i = 0; i < plugins.length; i++) {
    // Not every plugin in the ordering may be loaded at runtime.
    var possiblePlugin = plugins[i];
    if (possiblePlugin) {
      var extractedEvents = possiblePlugin.extractEvents(topLevelType, targetInst, nativeEvent, nativeEventTarget);
      if (extractedEvents) {
        events = accumulateInto(events, extractedEvents);
      }
    }
  }
  //此时的 event 含有_dispatchListeners 属性。。一个数组，里面有绑定的回调函数
  return events;
}

在 runEventsInBatch这里面先得到 eventQueue，也就是 target 为合成事件对象的 proxy 对象。然后对数组元素挨个进行 executeDispatchesInOrder，也就是对合成事件对象的 _dispatchListeners 按序执行。


紧接着~~~~~ 


//接下来就是这一段了
// Create a fake event type.
var evtType = 'react-' + (name ? name : 'invokeguardedcallback');

// Attach our event handlers
window.addEventListener('error', onError);
//sck 事件创建于触发
console.log('sck--', evtType);
//为甚好多次？ 
// sck-- react-invokeguardedcallback
debugger
fakeNode.addEventListener(evtType, callCallback, false);

// Synchronously dispatch our fake event. If the user-provided function
// errors, it will trigger our global error handler.
evt.initEvent(evtType, false, false);
fakeNode.dispatchEvent(evt);

```

dispatchInteractiveEvent->interactiveUpdates->runExtractedEventsInBatch->runEventsInBatch->循环执行所有绑定的事件


### 总结
React事件系统为了兼容各种版本的浏览器而做了大量工作，与原生事件不同的点，只在于React对事件进行统一而不是分散的存储与管理，捕获事件后内部生成合成事件提高浏览器的兼容度，执行回调函数后再进行销毁释放内存，从而大大提高网页的响应性能。