# R树
(转载)
R树是一种与B树类似的高度平衡树。这种索引是动态的，不需要定期重建。索引记录（Index Records）保存在叶节点中，索引记录包含指向数据对象的指针。注意原文中将索引的终端称为索引记录而不是空间对象。

空间数据库由一系列元组（tuple）构成，每个元组代表一个空间对象，并且每个元组都有一个唯一的标识符，标识符的作用是用于检索。

R树中的叶节点是以下面这种方式保存条目（entry）的：（I，tuple-identifier）。其中，I是一个n维矩形，表示空间对象的外廓矩形。Tuple-identifier是一个指向空间对象的指针。另外，原文中写到允许矩形I趋向于无穷大，却没有写到是否允许矩形I在某个维度上宽度为0。

R树中的非叶节点是这样保存条目的：（I，child-pointer）。其中，I是其所有子节点的外廓矩形的总外廓矩形。Child-pointer是指向下一级节点的指针。

原文中的表达方式似乎表明，节点中保存有多个条目，每个条目都有一个外廓矩形。我过去认为每个节点有一个外廓矩形及若干指针。但是从原文来看，我过去的理解是错误的。原文是这样的“Non-leaf nodes contain entries of the form （I，child-pointer） ”

记M为一个节点中的最大条目数。取m<=M/2，约定此m为一个节点中的最小条目数。此时，R树具有如下性质：

* （1）若叶节点不是根节点，则每个叶节点所包含的索引记录个数介于m与M之间
* （2）对于叶节点中的索引记录（I，tuple-identifier），I是其最小外包矩形。
* （3）若非叶节点不是根节点，则其中包含的子节点个数介于m与M之间。
* （4）对于非叶节点中的条目（I，child-pointer），I表示能够覆盖其所有子节点的外包矩形的外包矩形。注意原文中没有使用索引记录（index record）这个词，而用了条目（entry）。
* （5）根节点若非叶节点，则其至少有两个子节点。
* （6）所有叶子节点都在同一层。

一个有N个索引记录的R树，其高度最大为logm（N）-1，这是因为其每个节点至少要有m个记录。节点的总个数最多为N/m + N/m2 + …… + 1。最差的情况下，除根节点外的各节点空间利用率为m/M。通常情况下节点中的条目数不会只有m个，所以树的实际高度通常小于logm（N）-1，空间利用率通常高于m/M。如果节点中有超过3或4个条目，这个R树就会非常宽，从而导致绝大多数空间都用于保存叶节点，也就是保存索引记录。我无法理解原文是赞成这一现象还是反对这一现象。不同的m值会影响索引的性能。

## 1搜索算法
R树搜索算法与B树类似，都是从根节点向下搜索。然而，在搜索到某个节点后，接下来可能需要搜索这个节点下属的几个子树（而不是只搜索其中一个）。也就是说，搜索不是单向的。因此，在某些糟糕的情况下，R树可能无法保证性能。虽然如此，对于绝大多数的数据来说，R树的更新算法可以保证R树有一个相对较好的结构，能够保证在进行搜索的时候，无关的区域不会参与检索，只有在搜索区域附近的数据才会被检索（examine）。

下面描述搜索算法Search。我们将一个索引条目（Index Entry）记为E，其中的外廓矩形记为EI，其中的指针，不管它是叶节点中的指针还是中间节点中的指针，我们均将其记为Ep。下文均使用类似的约定。

算法Search：

记R树的根节点记为T。搜索算法要求输入一个搜索矩形S，输出所有与S相交的索引记录。

* 步骤S1：搜索子树——如果T不是一个叶节点，则检查其中的每一个条目E，如果EI与S相交，则对Ep所指向的那个子树根节点调用Search算法。这里注意，Search算法接收的输入为一个根节点，所以在描述算法的时候，原文称对子树根节点调用Search算法，而不是对子树调用Search算法。

* 步骤S2：搜索叶节点——如果T是一个叶节点，则检查其中的每个条目E，如果EI与S相交，则E就是需要返回的检索结果之一。

## 2插入算法
为一个新的数据元组而往R树中插入一个索引记录的方法与B树的插入类似，都是将索引记录加入叶节点当中，如果节点上溢就进行节点分裂。节点分裂有可能导致上一级的节点也发生分裂，分裂操作有可能一直向上传递。

#### 2.1算法Insert：

原文没有谈及算法的输入输出，只是说本算法要往R树里插入一个索引条目E。

* 步骤I1：为新记录寻找保存位置——调用算法ChooseLeaf，选择一个用于保存E的叶节点L。

* 步骤I2：将记录存入叶节点——如果节点L中有存储空间，则将E保存在里面。否则使用算法SplitNode执行节点分裂操作，节点L将变成两个新节点L和LL，L和LL中保存了E和旧L中的所有条目。

* 步骤I3：向上传递树的变化——对节点L调用算法AdjustTree。如果步骤I2中进行过节点分裂操作，那还要对LL调用算法AdjustTree。

* 步骤I4：树的长高——如果节点的分裂操作向上传递导致根节点分裂，那就要新建一个根节点。新的根节点的两个子节点就是旧子节点分裂后形成的两个节点。

#### 2.2算法ChooseLeaf

原文没有谈及输入输出。本算法选择一个叶节点用于保存新的索引条目E。
* 步骤CL1：初始化——记R树的根节点为N。

* 步骤CL2：检查叶节点——如果N是个叶节点，返回N

* 步骤CL3：选择子树——如果N不是叶节点，则从N中所有的条目中选出一个最佳的条目F，选择的标准是：如果E加入F后，F的外廓矩形FI扩张最小，则F就是最佳的条目。原文后面还有一句话Resolve ties by choosing the entry with the rectangle of smallest area。对于这句话我是这样理解的：如果有两个条目在加入E后外廓矩形的扩张程度相等，则在这两者中选择外廓矩形较小的那个。另外值得一说的是，由于节点结构是“节点包括若干条目，每个条目都有外廓矩形”，所以F是一个条目而不是一个节点；同样是由于这个原因，CL2必须在CL3之前执行，否则就会产生试图往一条索引记录里插入另一条索引记录的错误。

* 步骤CL4：向下寻找直至达到叶节点——记Fp指向的孩子节点为N，然后返回步骤CL2循环运算，直至查找到叶节点。

#### 2.3算法AdjustTree

由叶节点L向上运算直至根节点，目的是调整外廓矩形，并将节点分裂的结果向上传递。
* 步骤AT1：初始化——将L记为N。如果L已经被分裂成为了L和LL，则将LL记为NN。

* 步骤AT2：检查运算是否结束——如果N是根节点，算法中止。

* 步骤AT3：调整父节点中的外廓矩形（原文是Covering Rectangle），或曰，外廓矩形的调整——记P是N的父节点，记En是P里表示N的那个条目。调整EnI，使其恰好能够无冗余地包括N中的所有条目的外廓矩形。

* 步骤AT4：向上传递节点分裂的结果——如果由于节点分裂产生了一个新的节点NN。注意是节点NN不是条目NN。则新建一个条目Enn，其中的Ennp指向NN，EnnI是NN的最小外廓矩形。如果P有空间容纳Enn，则将Enn加入P当中。如果节点P已满，则对节点P执行算法SplitNode，生成新的P和PP，P和PP包括旧的P中的所有条目和条目Enn。

* 步骤AT5：上移——如果P发生了分裂，则令N=P，令NN=PP，然后返回步骤AT2重新开始运算。

#### 2.4算法SplitNode
这个算法非常复杂，下文将专门介绍。

#### 2.5总结
基本过程是这样的：先找到一个用于插入的位置。如果能直接插入，则直接插入，然后调整其父节点的外廓矩形。注意是父节点中的外廓矩形而不是它插入的那个节点的外廓矩形，这是由节点的存储结构决定的。

果插入不能直接插入，则进行节点分裂然后插入。这个比较复杂了。分裂后的一个节点会占用旧节点的位置，对这个节点要进行父节点调整。对于另外一个新节点，需要将其作为一个条目插入到原本的那个父节点中。这又是一个插入操作，又有可能产生节点分裂。但是在算法描述中这里不再调用算法Insert，而是直接从AT2重新计算，相当于重新调用算法AdjustTree。换言之，插入操作不是一个简单的递归或者循环，真正递归的是算法AdjustTree。另外，这究竟是递归还是循环尚不清楚。从算法描述上来说，这更接近循环。最后，如果有分裂的话，有可能导致根节点分裂，这又产生了一个新建根的问题。

这个算法描述的不清楚，如果想用代码将其实现的话还存在很多困难。

## 3删除算法
#### 3.1算法Delete
从R树中移除索引记录E

* 步骤D1：寻找包含记录的节点——调用算法FindLeaf来定位包含E的叶节点L。如果没有找到，则算法中止。
* 步骤D2：删除记录——将E从L中移除。
* 步骤D3：传递变化——调用算法CondenseTree，传入L（原文是Passing L）。
* 步骤D4：缩短树——如果树在经过调整后，根节点只剩下了一个子节点，那么就把这个子节点当成根节点。

#### 3.2算法FindLeaf

R树的根节点为T，查找包含索引条目E的叶节点

* 步骤FL1：查找子树——如果T不是叶节点，则逐个检查T中的每个条目F，看FI是否与EI相交。若相交，则对Fp指向的子树根节点调用算法FindLeaf。如此递归直至查找到E或者所有的条目都被检查过。按我的理解最后一句话有问题，第一，如果查找结果为空，则只需要查找根节点T中的所有条目F，不必遍历整个树；第二，步骤FL1不可能找到E，E保存在叶节点中，它肯定会在FL2中被找到。
* 步骤FL2：如果T是一个叶节点，则逐个检查T中的每一个条目看有没有E。原文是这样写的：Check each entry to see if it matches E。对于这个描述我有点困惑，因为这个算法没有说明什么叫it matches E。换言之，这个算法中的E是以什么方式传入的？不应该是一个地址吗？这样根本不需要检查it matches E，只有it is E或者it is not E两种情况。接着说算法。如果找到了E，返回T。

#### 3.3算法CondenseTree

树的压缩。叶节点L中刚刚删除了一个条目。现在要评估这个节点的条目数是否太少，如果太少，应将这些条目移到其他节点中。如果有必要，要逐级向上进行这种评估。调整向上传递的路径上的所有外廓矩形，使其变小。

* 步骤CT1：初始化——记N=L，定义Q为需要评估的节点数组，初始化的时候将此数组置空。
* 步骤CT2：查找父条目，注意是父条目，不是父节点——如果N是根节点，转到步骤CT6。如果N不是根节点，记P为N的父节点，并记En为P中代表N的那个条目。
* 步骤CT3：评估节点是否下溢——如果N中的条目数小于m，意味着节点N下溢，此时应当将En从P中移除，并将N加入Q。注意N是一个节点，他把一个节点放入Q了。
* 步骤CT4：调整外廓矩形——如果N没有被评估（我的理解是，如果N没有下溢），则调整En的外廓矩形EnI，使其尽量变小、恰好包含N中的所有条目。
* 步骤CT5：向上一层——令N=P，返回步骤CT2重新执行。
* 步骤CT6：重新插入孤立条目——对Q中所有节点的所有条目执行重新插入。叶节点中的条目使用算法Insert重新插入到树的叶节点中；较高层节点中的条目必须插入到树的较高位置上。这是为了保证这些较高层节点下的子树的叶子节点、与其他叶子节点能够放置在同一层上。这里需要注意，第一，Q中的都是节点，也就是说，Q中保存的是节点，如果节点不是叶节点，那Q中实际上保存了子树；第二，但是算法要求不是把Q重新插入进去（Q已经下溢了，已经被放弃了），而是把Q中的条目插入进去，如果这个条目是一个底层的索引条目，那可以调用算法Insert进行插入，但是如果这个条目是中间条目，本文没有描述清楚应该怎么插入，这一点很严重。

上文所描述的处理下溢节点的流程与B树中的相应操作不同。B树中对这个问题的处理方法是将两个或更多个相邻的节点合并起来。对于R树来说，虽然其中没有像B树中那样的“相邻节点”的概念，但R树仍然可以借用与B树类似的处理方法：下溢的节点与其某个兄弟节点合并，选择目标兄弟节点的原则是合并后面积的增加最小；或者，把节点中的各个条目分配入兄弟节点中。这两种模仿B树的解决方法都有可能导致节点的分裂。选择“重新插入”这一解决方法是出于两个原因。其一，这种方法同样可以完成任务，并且更容易实现。这种方法会有比较好的性能，因为重新插入操作所涉及的磁盘页通常就是前几个步骤的查找操作所涉及的磁盘页，它们在前几步的操作中就已经调入内存了。其二，重新插入这个操作进一步优化了树的空间结构。一个条目如果长期保存在同一个父节点中，有可能导致缓慢的索引性能恶化。重新插入的方法一定程度上避免了这个问题。

## 4更新和其他操作
如果一个数据元组的形状发生了更新，并因此导致此数据元组的外廓矩形发生变化，则相应的索引记录必须被删除、更新、然后重新插入。这是为了保证更新后的索引记录能够保存在树的正确位置上。

除上面所述的那种搜索操作外，下面几种搜索操作也比较常用。比如说，查找完全被某个区域所包含的数据对象，或者查找完全包含了某个区域的数据对象。这几种搜索都能够用前述搜索算法的直接变形（原文是straightforward variations）所实现。查找某个特定的条目的方法可以用删除操作的算法FindLeaf实现。批量删除，也就是说删除某个区域内所有数据对象的索引条目的方法，R树也能够提供良好的支持。但是原文没有说应该怎么做。

## 5节点分裂

如果想要往一个已经包含了M个条目的满节点中添加一个新条目，就需要将M+1个节点分别划分到两个节点当中。划分出的两个节点应该有尽可能大的区别，为了保证这一点，两个新节点都需要经过一系列的搜索操作的检查。由于进行搜索操作的时候，一个节点是否被访问取决于它的外廓矩形是否与查询范围相交，所以节点划分的原则是两个新节点的外廓矩形的总面积应当尽可能少。注意是总面积尽可能少而不是重叠面积尽可能少。

在使用算法ChooseLeaf来决定新的条目插入到哪里的时候，使用的也是类似的规则。算法ChooseLeaf选定的子树应当是那个加入新条目后外廓矩形的面积扩大程度最小的子树。

### 5.1第一，遍历算法

寻找最佳的分裂方案的最佳方法是生成所有可能的分组方案并从中选择最佳方案。然而可能的分组方案太多。我们实现了一种经过修改的遍历算法，以其作为评判其他算法的标准。但是这种算法在节点较大的时候运算速度很慢。

### 5.2第二，平方成本算法

这种算法尝试选择一种总面积最小的节点分裂方案，但本方法不保证得出的方案是最佳方案。这种方法的复杂性与M平方相关、与空间维度线性相关。这种算法首先从M+1个条目中选出两个条目作为两个新组的第一个元素。这两个条目的选择原则为：两个条目如果放在同一组中，会有最多的冗余空间。也就是说，两个条目的总的外廓矩形面积，减去两个条目所占用的面积，其差最大。接下来剩余的条目将按一定的顺序分入两个组当中。顺序是这样计算的：每个条目插入A组或B组导致的面积增长程度会有所不同。插入两个组后，两个组面积增长程度差异最大的那个条目、就是接下来要插入的条目。

#### 算法QuadraticSplit

* 步骤QS1：为两个组选择第一个条目——使用算法PickSeed来为两个组选择第一个元素，分别把选中的两个条目分配到两个组当中。
* 步骤QS2：检查是否已经完成分配——如果所有的条目都已经分配完毕，算法中止。如果一个组中现有的条目太少，以至于剩余未分配的所有条目都要分配到这个组当中，才能避免下溢，则将剩余的所有条目都分配到这个组中，算法中止。
* 步骤QS3：选择一个条目进行分配——调用算法PickNext来选择下一个进行分配的条目。将其加入一个组，要求它加入这个组比加入另一个组能有较小的面积增长。如果两个面积增长相同，则将其加入面积较小的那个组当中。若两个组面积也相同，则加入条目数更少的那个组当中。然后返回步骤QS2继续进行。

#### 算法PickSeeds

* 步骤PS1：计算分组方法的效率指数——对于每一对条目E1、E2和同时包含E1E2的外廓矩形J，计算d=Area（J）-Area（E1I）-Area（E2I）。注意它没有考虑图形重叠的问题。
* 步骤PS2：选择浪费程度最大的一组——选择d值最大的一组E1和E2作为返回值。

#### 算法PickNext
* 步骤PN1：将每个条目分入每个组的成本——对于每个尚未分组的条目，计算d1=将此条目加入第一组后导致的外廓矩形面积增张。同理计算d2。
* 步骤PN2：选择用于分配的最佳条目——选择d1与d2之差最大的那个条目。

### 5.3第三，线性成本算法

这个算法与M和维度数均线性相关。本算法的算法LinearSplit与平方成本算法的算法QuadraticSplit相同。算法PickNext直接选择任意一条未分组的条目。

#### 算法LinerPickSeed

本来我没有看明白这个算法。后来帆风同学告诉我，Normalize这个词可以翻译成“标准化”。这样一来我似乎明白了这个算法所表达的意思。于是我觉得应该是下面这样翻译。

* 步骤LPS1：选择各维度的极端矩形——对于各维度，找到外廓矩形下界最大的那个条目和外廓矩形上界最小的那个条目（原文是find the entry whose rectangle has the highest low side, and the one with the lowest high side）。记住二者之间的距离（原文是Separation）。

* 步骤LPS2：根据矩形簇的形状进行调整——对前述划分进行标准化。标准化的方式是：对于每一个维度，步骤LPS1计算出了一个距离，不妨记为S；又，对于每个维度，对所有条目的外廓矩形构造一个总的大外廓矩形，记这个大外廓矩形在相应维度上的投影为L；S除以L即为相应维度标准化后的距离。原文是这样的：Normalize the separations by dividing by the width of the entire set along the corresponding dimension。这句话实在太难表达了。

* 步骤LPS3：选择标准化距离最大的一组——原文是这样：Choose the pair with the greatest normalized separation along any dimension。

# Golang实现

实现了搜索、插入操作,删除操作暂未实现，代码从一个C版本转的

demo选取了北京8万个坐标点构建一棵r-tree

### Run Demo
    
    * 修改point_data_path、web_index_path路径
    * 修改httpServer地址及端口，同时对应修改web/index.html中的:"http://10.94.106.47:9999/search"
    * go build -o rtree_server
    * ./rtree_server
    * 
    * 浏览器打开:http://HOST:9999/index


