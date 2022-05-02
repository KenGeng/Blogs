# The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing 阅读笔记

*论文发表时间：2015 作者机构：谷歌 链接：[link](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/43864.pdf)*
## TL;DR
Google提出Dataflow模型来重新思考流计算领域中对大规模无界、不保证有序的数据的处理。核心思路是针对event time，厘清Window的语义(Fixed/Sliding/Sessions)与操作(Assignment/Merging)；针对processing time，定义不同的trigger策略(e.g. watermark)，以及处理同一window多个视图的策略(Discarding/Accumulating/Accumulating and Retracting)。文章中给出的例子非常详细，可以作为教材看了。

## 重要内容记录
### 论文贡献
1. 指出在一个无界(unbounded)无序(unordered)数据源(source)上如何根据数据自身特征设置窗口(window)并进行基于事件时间(event-time)顺序的计算。 计算过程中的正确性、延迟、花费可以根据需求进行权衡。
2. 将计算的pipeline抽象为四个部分：
	1. What results are being computed -> 要算什么(由用户决定)
	2. Where in event time they are being computed. -> 取哪一部分的event time内的数据进行计算(window策略)
	3. When in processing time they are materialized. -> 在processing time的哪一时刻将计算结果物化(trigger策略-watermark)
	4. How earlier results relate to later refinements. -> 如何merge之前计算的结果和迟到的数据
3. 将数据处理的逻辑概念与物理实现(batch/micro batch/streaming)解耦 -> 批流一体的计算模型
另外Dataflow模型理论上并不"magic"，它并没有拓展batch/micro batch/streaming模型的能力，但是它使得构建大型的数据处理pipeline的时候可以以更统一的视角去考虑latency/correctness/cost这些因素。
### 概念定义
#### Unbounded/Bounded
unbounded data sets: infinite data sets

bounded data sets: finite data sets

上述两种数据，streaming或者batch计算引擎实际中都可以处理(e.g. repeated runs of batch systems on infinite data)

#### Windowing
Windowing slices up a dataset into finite chunks for processing as a group.
window是基于时间(物理时间或者逻辑时间戳)来定义的

Aligned windows: applied across **all** the data for the window of time in question

Unaligned windows: applied across only specific **subsets** of the data (e.g. per key) for the given window of time.

![[Pasted image 20220502104315.png]]
##### Fixed Window
Fixed windows (sometimes called tumbling windows) are defined by a static window size, e.g. hourly windows or daily windows. *typically aligned.*

##### Sliding Window
Sliding windows are defined by a window size and slide period, e.g. hourly windows starting every minute. *typically aligned.* Fixed windows are really a special case of sliding windows where size equals period.

##### Session Window
Sessions are windows that capture some period of activity over a subset of the data, in this case per key. Typically they are defined by a timeout gap. Any events that occur within a span of time less than the timeout are grouped together as a session. Sessions are unaligned windows.

#### Time Domains
##### Event Time
the time at which the event itself actually occurred, i.e. a record of system clock time (for whatever system generated the event) at the time of occurrence. 
##### Processing Time
the time at which an event is observed at any given point during processing within the pipeline, i.e. the current time according to the system clock.

在实际系统中，由于处理的延迟或者调度算法，event time和processing timet 通常是不一致的(skew)。

这种skew可以借助watermark(Global progress metrics)来可视化。watermark, which is a lower bound (often heuristically established) on event times that have been processed by the pipeline.

Watermark一般慢于event time

![[Pasted image 20220502110445.png]]

### Dataflow模型
回顾batch模型，可以认为它有两个原语(primitives)：
1. *ParDo*: element-wise; apply a user-defined function on each input and then yield zero of more output element per input. 这个operation可以很容易地从bounded data迁移到unbounded data
2. *GroupByKey*: collects all data for a given key before sending them downstream for reduction. 这个operation如果想在unbounded data上应用，那我们需要定义window

#### Windowing
*GroupByKey -> GroupByKeyAndWindow*
*(key, value) -> (key, value, event time, window)*
由于aligned window可以看作是unaligned window的特例，所以接下来只考虑unaligned window。

Window操作可以分为两步：
1. AssignWindow：将element分配到0或多个window
2. MergeWindow：合并两个window。这个操作使得我们可以构建基于数据(data-driven)的window

![[Pasted image 20220502114611.png]]

![[Pasted image 20220502114623.png]]
#### Triggers & Incremental Processing
我们需要判断什么时候window完成了从而触发计算，一个常见的解决方案是watermark(global event-time progress metric)，但watermark有一些缺点：
1. 有时候太快：有可能有late element在某个watermark之后到达，导致window内的数据不完全
2. 有时候太慢：由于input source的系统延迟，watermark可能前进得比较慢，导致晚于实际的event time较多，进而导致触发window生成结果的latency很大

所以单纯的watermark是不可靠的，论文提出的解决思路是借鉴mini batch,对某个window，基于trigger，允许产生多个视图(论文用的单词是pane)。即Windowing determines where in event time data are grouped together for processing. Triggering determines when in processing time the results of groupings are emitted as panes.
Dataflow支持的trigger包括基于watermark(or percentile watermark(完成一定比例的数据就触发))的和基于数据的(counts, bytes, pattern matching...)，这些trigger也可以组合。

一个重要的点是，我们如何处理同一个window的不同视图的关系。论文定义了3种策略：
1. Discarding(丢弃)：当一个window的计算被触发时，之前的计算结果(上个视图)被丢弃，当前的计算结果发给下游。这依赖于window内数据的独立性
2. Accumulating(累加)：当一个window触发时，之前的计算结果保留，并且将当前计算结果“添加”到之前的计算结果中发送给下游。注意这里的“添加”操作本质是一种更新操作
3. Accumulating & Retracting：当一个window触发时，之前的计算结果保留，先将之前的计算结果作为retraction发送给下游，再将当前计算结果“添加”到之前的计算结果中发送给下游。这种策略主要用于多个串行的grouping window。
![[Pasted image 20220501122604.png]]
### 例子
数据特征：
![[Pasted image 20220502122447.png]]
注意蜿蜒的虚线是实际的watermark。以元素7为例，它真实的event time是12:02:30左右，它的processing time是12:05:40左右，当processing time为12:07:30左右的时候，watermark才前进到12:04。


不同计算引擎的处理：
注意阴影面积代表window的一个视图
window计算的触发沿processing time轴依次发生
沿着event time轴，多个window产生
The watermark trigger fires when the watermark passes the end of the window in question.

![[Pasted image 20220502122457.png]]

![[Pasted image 20220502122651.png]]

![[Pasted image 20220502122742.png]]

![[Pasted image 20220502122811.png]]

![[Pasted image 20220502122922.png]]
![[Pasted image 20220502122934.png]]

![[Pasted image 20220502123048.png]]

![[Pasted image 20220502123145.png]]

注意，最左边的窗口在late element到达时，重算了。另外相比前面的micro batch，可以看到latency要大，例如第二个窗口(7,8,3,4)在watermark在processing time为12:07:30左右超过event time(12:04)时才触发。

一个简单的优化是添加定时的触发：
![[Pasted image 20220502135717.png]]

最复杂的session window(注意window会基于timeout合并)：
![[Pasted image 20220502135910.png]]

### 典型场景
1. Large Scale Backfills & The Lambda Architecture: Unified Model
2. Unaligned Windows: Sessions(谷歌内部搜索、推荐要用，该论文主要目的场景)
3. Billing: Triggers, Accumulation, & Retraction
4. Statistics Calculation: Watermark Triggers； Abuse detection using percentile watermarks(快速比严格的准确性更重要)
5. Recommendations: Processing Time Triggers；实时推荐
6. Anomaly Detection: Data-Driven & Composite Triggers；异常检测(实时风控)


## 解决了什么问题
针对无界数据集，如何用一个批流一体的模型来进行处理。
## 用了什么思路
基于window和trigger，提出Dataflow模型，并且定义了如何merge window。
## 效果如何
Google Dataflow产品，后续流计算引擎基本都借鉴了Dataflow。
## Anecdote
1. Flink原型时间(2014)早于Dataflow(2015)
2. 这个谷歌员工的[分享](https://www.youtube.com/watch?v=3UfZN59Nsk8&ab_channel=%40Scale )也对理解论文有一些帮助。视频里谷歌在2015年推出的Dataflow产品UI就挺好的了，谷歌还是厉害。
3. watermark和event time的关系让我花了一些时间理解，之前先入为主地以为watermark就是event time，但在分析论文example的时候遇到了无法解释的问题。感谢作者写得非常细致，我仔细读了前文后才理解了文中的watermark是global event-time的近似，而不是某个具体数据点的event time.

*祝生活愉快 By Ken G*




