# The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing 阅读笔记

*论文发表时间：2015 作者机构：谷歌 链接：[link](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/43864.pdf)*
## TL;DR
Google提出Dataflow模型来重新思考流计算领域中对大规模无界、无序的数据的处理。核心思路是针对event time，厘清Window的语义(Fixed/Sliding/Sessions)与操作(Assignment/Merging)；针对processing time，定义不同的trigger策略(e.g. watermark)，以及处理同一window多个视图的策略(Discarding/Accumulating/Accumulating and Retracting)。文章中给出的例子非常详细，可以作为教材看了。

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

1. Flink原型时间(2014)早于Dataflow(2015)
2. 这个谷歌员工的[分享](https://www.youtube.com/watch?v=3UfZN59Nsk8&ab_channel=%40Scale )也对理解论文有一些帮助。视频里谷歌在2015年推出的Dataflow产品UI就挺好的了，谷歌还是厉害。
3. watermark和event time的关系让我花了一些时间理解，之前先入为主地以为watermark就是event time，但在分析论文example的时候遇到了无法解释的问题。感谢作者写得非常细致，我仔细读了前文后才理解了文中的watermark是global event-time的近似，而不是某个具体数据点的event time.

*祝生活愉快 By Ken G*



