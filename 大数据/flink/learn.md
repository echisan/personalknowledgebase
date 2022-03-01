https://nightlies.apache.org/flink/flink-docs-release-1.13/zh/docs/learn-flink/datastream_api/

EventTime与Watermarks

## EventTime

EventTime需要给flink提供一个时间戳提取器和Watermark生成器，Flink 将使用它们来跟踪事件时间的进度

## Watermarks

 *watermarks 的作用* — 它们定义何时停止等待较早的事件。

Flink 中事件时间的处理取决于 *watermark 生成器*，后者将带有时间戳的特殊元素插入流中形成 *watermarks*。事件时间 *t* 的 watermark 代表 *t* 之前（很可能）都已经到达。

## 延迟 VS 正确性

watermarks 给了开发者流处理的一种选择，它们使开发人员在开发应用程序时可以控制延迟和完整性之间的权衡。

我们可以把 watermarks 的边界时间配置的相对较短，从而冒着在输入了解不完全的情况下产生结果的风险-即可能会很快产生错误结果。或者，你可以等待更长的时间，并利用对输入流的更全面的了解来产生结果。

当然也可以实施混合解决方案，先快速产生初步结果，然后在处理其他（最新）数据时向这些结果提供更新。对于有一些对延迟的容忍程度很低，但是又对结果有很严格的要求的场景下，或许是一个福音。

## 延迟

延迟是相对于 watermarks 定义的。`Watermark(t)` 表示事件流的时间已经到达了 *t*; watermark 之后的时间戳 ≤ *t* 的任何事件都被称之为延迟事件。

