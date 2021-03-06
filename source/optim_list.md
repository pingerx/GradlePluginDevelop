#### 优化开始，Are you ready?

##### 优化分析工具

  - Hierarchy View 在Android SDK里自带，常用来查看界面的视图结构是否过于复杂，用于了解哪些视图过度绘制，又该如何进行改进。
  - Lint 是 ADT 自带的静态代码扫描工具，可以给 XML 布局文件和 项目代码中不合理的或存在风险的模块提出改善性建议。官方关于 Lint 的实际使用的提示，列举几点如下：
      包含无用的分支，建议去除；
      包含无用的父控件，建议去除；
      警告该布局深度过深；
      建议使用 compound drawables ；
      建议使用 merge 标签；
  - Systrace 在Android DDMS 里自带，可以用来跟踪 graphics 、view 和 window 的信息，发现一些深层次的问题。(用起来比较复杂)

  - Track 在 Android DDMS里自带，是个很棒的用来跟踪构造视图的时候哪些方法费时，精确到每一个函数，无论是应用函数还是系统函数，我们可以很容易地看到掉帧的地方以及那一帧所有函数的调用情况，找出问题点进行优化。
  
  - OverDraw 
     通过在 Android 设备的设置 APP 的开发者选项里打开 “ 调试 GPU 过度绘制 ” ，来查看应用所有界面及分支界面下的过度绘制情况，方便进行优化。
     
  - GPU 呈现模式分析   
     通过在 Android 设备的设置 APP 的开发者选项里启动 “ GPU 呈现模式分析 ” ，可以得到最近 128 帧 每一帧渲染的时间，分析性能渲染的性能及性能瓶颈。
     
  - StrictMode
     
     通过在 Android 设备的设置 APP 的开发者选项里启动 “ 严格模式 ” ，来查看应用哪些操作在主线程上执行时间过长。当一些操作违背了严格模式时屏幕的四周边界会闪烁红色，同时输出 StrictMode 的相关信息到 LOGCAT 日志中。
     
  - Animator duration scale
     
     通过在 Android 设备的设置 APP 的开发者选项里打开 “ 窗口动画缩放 ” / “ 过渡动画缩放 ” / “ 动画程序时长缩放 ”，来加速或减慢动画的时间，以查看加速或减慢状态下的动画是否会有问题。
     
  - Show hardware layer updates
     
     通过在 Android 设备的设置 APP 的开发者选项里启动 “ 显示硬件层更新 ”，当 Flash 硬件层在进行更新时会显示为绿色。使用这个工具可以让你查看在动画期间哪些不期望更新的布局有更新，方便你进行优化，以获得应用更好的性能。实例《 Optimizing Android Hardware Layers 》（需要翻墙）:「 戳我 」。
     
##### 优化方法

1 优化布局的结构

- 布局结构太复杂，会减慢渲染的速度，造成性能瓶颈。我们可以通过以下这些惯用、有效的布局原则来优化：
- 避免复杂的View层级。布局越复杂就越臃肿，就越容易出现性能问题，寻找最节省资源的方式去展示嵌套的内容；
- 尽量避免在视图层级的顶层使用相对布局 RelativeLayout 。相对布局 RelativeLayout 比较耗资源，因为一个相对布局 RelativeLayout 需要两次度量来确保自己处理了所有的布局关系，而且这个问题会伴随着视图层级中的相对布局 RelativeLayout 的增多，而变得更严重；
- 布局层级一样的情况建议使用线性布局 LinearLayout 代替相对布局 RelativeLayout*，因为线性布局 LinearLayout 性能要更高一些；确实需要对分支进行相对布局 RelativeLayout 的时候，可以考虑更优化的网格布局 GridLayout ，它已经预处理了分支视图的关系，可以避免两次度量的问题；
- 相对复杂的布局建议采用相对布局 RelativeLayout ，相对布局 RelativeLayout 可以简单实现线性布局 LinearLayout 嵌套才能实现的布局；
- 不要使用绝对布局 AbsoluteLayout ；
- 将可重复使用的组件抽取出来并用 </include> 标签进行重用。如果应用多个地方的 UI 用到某个布局，就将其写成一个布局部件，便于各个 UI 重用。
- 使用 merge 标签减少布局的嵌套层次
- 去掉多余的不可见背景。有多层背景颜色的布局，只留最上层的对用户可见的颜色即可，其他用户不可见的底层颜色可以去掉，减少无效的绘制操作；
- 尽量避免使用 layout_weight 属性。使用包含 layout_weight 属性的线性布局 LinearLayout 每一个子组件都需要被测量两次，会消耗过多的系统资源。在使用 ListView 标签与 GridView 标签的时候，这个问题显的尤其重要，因为子组件会重复被创建。平分布局可以使用相对布局 RelativeLayout 里一个 0dp 的 view 做分割线来搞定，如果不行，那就……；
- 合理的界面的布局结构应是宽而浅，而不是窄而深；


    LinearLayout 在有weight属性时，为什么是可能会导致 2次measure ?
    
    分析源码发现，并不是所有的layout_weight都会导致两次measure：
    
    Vertical模式下，child设置了weight（height＝0，weight > 0）时将会跳过这一次Measure，之后会再一次Measure
    
    //Vertical模式下，child设置（height＝0，weight > 0）时将会跳过这一次Measure，之后会再一次Measure
    if (heightMode == MeasureSpec.EXACTLY && lp.height == 0 && lp.weight > 0) {
       // Optimization: don't bother measuring children who are going to use
       // leftover space. These views will get measured again down below if
       // there is any leftover space.
       final int totalLength = mTotalLength;
       mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
       skippedMeasure = true;//跳过这一次measure
    } 

2 优化处理逻辑

- 按需载入视图。某些不怎么重用的耗资源视图，可以等到需要的时候再加载，提高UI渲染速度；
- 使用 ViewStub 标签来加载一些不常用的布局；
- 动态地 inflation view 性能要比用 ViewStub 标签的 setVisiblity 性能要好，当然某些功能的实现采用 ViewStub 标签更合适；
- 尽量避免不必要的耗资源操作，节省宝贵的运算时间；
- 避免在 UI 线程进行繁重的操作。耗资源的操作（比如 IO 操作、网络操作、SQL 操作、列表刷新等）耗资源的操作应用后台进程去实现，不能占用 UI 线程，UI 线程是主线程，主线程是保持程序流畅的关键，应该只操作那些核心的 UI 操作，比如处理视图的属性和绘制；
- 最小化唤醒机制。我们常用广播来接收那些期望响应的消息和事件，但过多的响应超过本身需求的话，会消耗多余的 Android 设备性能和资源。所以应该最小化唤醒机制，当应用不关心这些消失和事件时，就关闭广播，并慎重选择那些要响应的 Intent 。
- 为低端设备考虑，比如 512M 内存、双核 CPU 、低分辨率，确保你的应用可以满足不同水平的设备。
- 优化应用的启动速度。当应用启动一个应用时，界面的尽快反馈显示可以给用户一个良好的体验。为了启动更快，可以延迟加载一些 UI 以及避免在应用 Application 层级初始化代码。

3 善用 DEBUG 工具

- 多使用Android提供的一些调试工具去追踪应用主要功能的性能情况；
- 多使用Android提供的一些调试工具去追踪应用主要功能的内存分配情况