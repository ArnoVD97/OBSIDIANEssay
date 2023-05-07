在写Masonry之前，我想先来聊聊约束的基础知识，我们首先要了解一个View的约束需要确定的是两个因素，一个是宽高信息，另外一个是位置信息。 只有确定这两个因素才能真正的确定一个View的约束，否则约束会爆警告。不管你怎么加约束，其实最后归根到底都是确实的这两个信息，那么我们了解这个有什么好处呢？我们可以通过约束转化来了解我们多添加了约束，是否缺失了某个约束，这种思想可以帮助我们快速查询问题所在。

但是有很多童鞋会发现在使用 Masonry 的时候，如果控件是UILabel，UIImageView，UIButton等这些组件及某些包含它们的系统组件只需要指定控件的位置约束，根本不需要指定宽高约束即可完成布局任务，这是为什么呢？这是因为这些控件中有 intrinsicContentSize 这个属性，intrinsicContentSize的作用其实很简单，它会自己根据内容计算出控件的固有宽高，在布局过程当你不指定宽高约束的时候，它就会生效。具体的内容我会在下面说到。这里就不过多叙述了。
首先，Masonry的添加布局主要有三个，三个方法的作用分别是创建约束；更新某个约束，其他约束不变；移除先前所有约束，添加新到的约束。这三个方法根据场景需要合理使用，否则可能造成内存问题，优化方式下面我们会来聊一下，这里就不过多叙述了。
```java
- (NSArray *)mas_makeConstraints:(void(NS_NOESCAPE ^)(MASConstraintMaker *make))block;

- (NSArray *)mas_updateConstraints:(void(NS_NOESCAPE ^)(MASConstraintMaker *make))block;

- (NSArray *)mas_remakeConstraints:(void(NS_NOESCAPE ^)(MASConstraintMaker *make))block;

```
如果我们想设置一个具体的数值该怎么办呢？例如宽度我们想设置成10个单位，我们就可以如下设置。
```c
[subView mas_makeConstraints:^(MASConstraintMaker *make) {
   make.width.equalTo(@10);
}];

```

我们发现上面代码还有个问题，那就是 **equalTo** 这个函数的参数必须是一个对象类型，这就很尴尬了，为啥，书写太麻烦，这时候我们可以使用 **mas_equalTo** 这个函数，示例如下所示。
```c
[subView mas_makeConstraints:^(MASConstraintMaker *make) {
   make.width.mas_equalTo(10);
}];

```
上面书写的好像也不是一个最优的方案，虽然我们解决了后面的问题，但是前面的代码字母数又多了(懒癌发作😂)，这时候我们可以在我们的文件之前加上一个 **#define MAS_SHORTHAND_GLOBALS** 这样的[宏定义](https://so.csdn.net/so/search?q=%E5%AE%8F%E5%AE%9A%E4%B9%89&spm=1001.2101.3001.7020)，就可以直接使用equalTo(10)了，如下所示。
```c
#define MAS_SHORTHAND_GLOBALS

[subView mas_makeConstraints:^(MASConstraintMaker *make) {
   make.width.equalTo(10);
}];

```

如果我们想要设置subView的宽度等于父视图的宽度的50%，这时候我们该怎么编写我们的约束呢？我们可以用到 **multipliedBy** 和 **dividedBy**这两个方法，一个是乘法，一个是除法，较为简单，示例代码如下所示。
```c
//乘法
[subView mas_makeConstraints:^(MASConstraintMaker *make) {
   make.width.equalTo(self).multipliedBy(0.5);
}];
//除法
[subView mas_makeConstraints:^(MASConstraintMaker *make) {
   make.width.equalTo(self).dividedBy(2);
}];

```
如果我们想要subView的宽度等于高度的2倍，这时候该怎么办呢？我们需要指定equalTo()里面具体的值(实际上是传一个 MASViewAttribute 对象)，而不是简单的传一个控件对象，示例代码如下所示。
```c
[subView mas_makeConstraints:^(MASConstraintMaker *make) {
   make.width.equalTo(subView.mas_height).multipliedBy(2);
}];

```
我们就简单叙述一下约束优先级设置，Masonry为我们提供了三个优先级的方法，priorityLow()、priorityMedium()、priorityHigh()，这三个方法内部对应着不同的默认优先级，当然我们也可以使用priority() 设置具体的数值。示例代码如下所示，关于约束优先级具体使用也会在后面的模块中说到。
```c
[subView mas_makeConstraints:^(MASConstraintMaker *make) {
   make.width.equalTo(subView4).priorityLow();
   make.width.equalTo(@[subView2, subView3,@100]).priorityHigh();
   make.width.equalTo(@300).priority(888);
}];

```
如果我们想让subView的宽度是 父视图的宽度的30% + 10个单位长度，这时候我们该怎么设置呢？其实这时候有点类似于 CSS 中的 calc() 函数，我们肯定不能设置两条约束条件，如果那样设置了，后面的约束条件就会把前面的约束条件给覆盖掉，对此我们如下设置即可。(offset方法和multipliedBy方法顺序无影响)
```c
[subView mas_makeConstraints:^(MASConstraintMaker *make) {
  make.width.equalTo(self).offset(10).multipliedBy(0.3);
}];

```

## 约束优先级 以及 intrinsicContentSize相关问题
约束优先级以及intrinsicContentSize的相关问题是我们不得不提到的问题.

首先来说一下为什么要有约束优先级,我们给定一个场景,假设我们设置在一个superView(宽度为 200)中的一个View子视图的左右边距都为0,然后第二个约束是视图的宽度为100,这时候就会出现问题,因为如果左右边距都为0,那么视图宽度为200,这样和第二个约束条件就发生了冲突,系统是不允许这样的问题出现的.那么我们想不在删除约束的情况下,该如何解决这种问题呢?这时候我们就需要通过设置约束优先级来解决这一类问题,系统通过比较两个”相互冲突的约束”的优先级，从而忽略低优先级的某个约束，达到正确布局的目的约束优先级默认都是1000.所以我们给设定一个根据具体情况设置一个合适的值即可,代码如下所示.
```c
[subView mas_makeConstraints:^(MASConstraintMaker *make) {
   make.left.right.equalTo(self);
   make.width.equalTo(@100).priority(888);
}];

```
约束优先级主要是应对与单个视图中多个约束发生冲突的时候解决问题的方案.而 intrinsicContentSize 主要应对于多个视图约束发生冲突的解决方案,我们就对着具体的实例来进行分析.

在最前面我们说到 在AutoLayout中， intrinsicContentSize的作用其实很简单，它会自己根据内容计算出控件的固有宽高，在布局过程当你不指定宽高约束的时候，它就会生效。

这个属性是非常的好用,但是也会出现对应的问题.例如我们现在有两个Label,两个Lable的约束条件如下所示.
```c
[label1 mas_makeConstraints:^(MASConstraintMaker *make) {
   make.left.equalTo(superView);
   make.top.equalTo(superView);
}];

[label2 mas_makeConstraints:^(MASConstraintMaker *make) {
   make.left.equalTo(label1.mas_right);
   make.top.equalTo(superView);
}];

```
上面的情况完全没有任何的问题,因为 intrinsicContentSize 属性的原因,我们轻松完成布局任务,但是当我们给 label2 添加一个 右边距等于superView.代码如下所示.
```c
[label2 mas_makeConstraints:^(MASConstraintMaker *make) {
   make.left.equalTo(label1.mas_right);
   make.right.equalTo(superView);
   make.top.equalTo(superView);
}];
```
这时候就会出现问题,label1 和 label2 必然有一个不能满足 intrinsicContentSize 约束条件,必然有一个需要拉伸才能完成约束布局任务,我们称这种问题叫做 Intrinsic冲突.

解决 Intrinsic冲突 一共有两种方案,一种是直接指定冲突的label 1 和 label 2的宽高约束信息.第二种就是利用 content Hugging／content Compression Resistance.原始方法如下所示.
```c
- (void)setContentHuggingPriority:(UILayoutPriority)priority forAxis:(UILayoutConstraintAxis)axis API_AVAILABLE(ios(6.0));
- (void)setContentCompressionResistancePriority:(UILayoutPriority)priority forAxis:(UILayoutConstraintAxis)axis API_AVAILABLE(ios(6.0));

```
Content Hugging 约束（不想变大约束）表示：如果组件的此属性优先级比另一个组件此属性优先级高的话，那么这个组件就保持不变，另一个可以在需要拉伸的时候拉伸。属性分横向和纵向2个方向。

Content Compression Resistance 约束（不想变小约束）表示：如果组件的此属性优先级比另一个组件此属性优先级高的话，那么这个组件就保持不变，另一个可以在需要压缩的时候压缩。属性分横向和纵向2个方向。 意思很明显。上面UIlabel这个例子中，很显然，如果某个UILabel使用Intrinsic Content Size的时候，另一个需要拉伸。 所以我们需要调整两个UILabel的 Content Hugging约束的优先级就可以啦。
所以我们可以通过设置这两个方法来解决 **Intrinsic冲突** 问题,假设 我们想让 label 2 拉伸,label1尽量不拉伸,我们就可以设置如下代码.具体代码如下所示.
```c
[label1  setContentHuggingPriority:UILayoutPriorityRequired forAxis:UILayoutConstraintAxisHorizontal];
[label2  setContentHuggingPriority:UILayoutPriorityDefaultLow forAxis:UILayoutConstraintAxisHorizontal];

```
# Masonry:iOS12
我们都知道，其实Masonry是封装系统的NSLayoutConstraints,简化了代码，但是在iOS12之前NSLayoutConstraints存在着致命的问题，那就是性能问题，其实这个在iOS12之后也会存在只是小了很多。那么在iOS12之前到底是什么原因导致这些问题呢？接下来我们逐一分析各种情况。

AutoLayout使用的布局算法其实是 Cassowary，在WWDC2018，官方对其性能问题提出了说明，如下图所示，我们可以清楚的看到iOS12前的AutoLayout布局性能是成指数性增长的。
![[Pasted image 20230507163808.png]]
但是不是所有的布局都有这样的问题呢？答案当然是否定的，如下图所示。所以说AutoLayout只是在某些情况存在着问题。
![[Pasted image 20230507163843.png]]
那么真正的原始是什么呢？因为iOS12之前，当有约束变化时都会重新创建一个计算引擎 NSISEngier 将约束关系重新加起来，重新计算。涉及到约束关系变多时，新的计算引擎需要重新计算，最终导致计算量指数级增加。

iOS12的AutoLayout更多的利用了Cassowary算法的界面更新策略，使其真正完成了高效的界面线性策略计算。使其尽量成线程增加，减少性能问题，最后允许我唠叨一句，讲真的，性能再强也是干不过Frame布局方式的，但是胜在简单方便。
# Masonry源码解析
![在这里插入图片描述](https://img-blog.csdnimg.cn/8211735c83604e79ae32da6a3adfed53.png#pic_center)

Masonry 主要方法由上述例子就可一窥全貌。Masonry主要通过对 `UIView`（`NSView`）、`NSArray`、`UIViewController` 进行分类扩展，从而提供自动布局的构建方法。
对于一个一般的布局约束，研究一下约束方法
```c
   [_masParView mas_makeConstraints:^(MASConstraintMaker *make) {

        make.edges.equalTo(self.view).with.insets(UIEdgeInsetsMake(100, 100, 100, 100));

    }];
```
调用的方法是`UIView`的分类`MASAdditions`中的`mas_makeConstraints:`方法
```c
/**
 *  Creates a MASConstraintMaker with the callee view.
 *  Any constraints defined are added to the view or the appropriate superview once the block has finished executing
 *  **@param** block scope within which you can build up the constraints which you wish to apply to the view.
 *  **@return** Array of created MASConstraints
 *  使用被调用的视图创建一个MASConstraintMaker。一旦代码块执行完毕，任何定义的约束
 *  都会被添加到视图或适当的父视图中
 *  @参数 块作用域，您可以在其中构建您希望应用于视图的约束。
 *  @返回 创建的MASConstraints的数组
 */

- (NSArray *)mas_makeConstraints:(**void**(NS_NOESCAPE ^)(MASConstraintMaker *make))block;
```
实现
```c
- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *))block {

    self.translatesAutoresizingMaskIntoConstraints = NO;

    MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];

    block(constraintMaker);

    return [constraintMaker install];

}
```
该方法会创建一个 `MASConstraintMaker` 对象，并将当前视图作为其初始化参数。然后，该方法将传入的 `block` 块作为参数传递给 `MASConstraintMaker` 对象，并调用其中的方法来描述视图的约束。最后，该方法调用 `install` 方法，将约束应用到当前视图上，并返回一个包含所有约束的数组。

首先，该方法将 `translatesAutoresizingMaskIntoConstraints` 属性设置为 `NO`，这是为了确保使用自动布局约束而非默认的 frame 布局方式。

然后，该方法创建了一个 `MASConstraintMaker` 对象，并将当前视图作为其初始化参数。接着，该方法将传入的 `block` 块作为参数传递给 `MASConstraintMaker` 对象，从而描述了视图之间的约束关系。这个 `block` 块参数中的 `MASConstraintMaker` 对象包含了各种用于描述视图约束的方法。在调用 `block` 块时，我们可以使用这些方法来描述当前视图与其它视图之间的相对位置、尺寸等信息。

最后，该方法调用 `install` 方法，将约束应用到当前视图上，并返回一个包含所有约束的数组。`install` 方法会根据之前设置的约束信息计算出合适的约束，并将其添加到相应的视图上。在这个方法返回之后，当前视图就会按照之前设置的约束进行自动布局。
```c
- (id)initWithView:(MAS_VIEW *)view {
    self = [super init];
    if (!self) return nil;
    
    self.view = view;
    self.constraints = NSMutableArray.new;
    
    return self;
}
```
该方法用于创建一个 `MASConstraintMaker` 对象，该对象包含了用于描述视图约束的方法。在使用 Masonry 自动布局库时，我们通常需要先创建一个 `MASConstraintMaker` 对象，然后使用其中的方法描述视图之间的约束关系，最后调用 `install` 方法将约束应用到相应的视图上。

在这个方法中，我们首先调用父类 `NSObject` 的 `init` 方法进行初始化，如果初始化失败，则直接返回 `nil`。接着，我们将传入的 `view` 参数保存到 `self.view` 属性中。这个 `view` 参数指的是我们需要设置约束的视图对象。

我们还创建了一个 `NSMutableArray` 类型的数组，用于保存后续添加的约束。我们将这个数组保存到 `self.constraints` 属性中。

最后，该方法返回初始化后的 `MASConstraintMaker` 对象。这个对象包含了各种用于描述视图约束的方法，我们可以在之后的代码中使用这些方法来描述视图之间的约束关系。


在 `mas_makeConstraints` 方法中传递的块参数是一个 block，这个 block 的参数类型是 `MASConstraintMaker` 对象，即一个用于描述视图约束的对象。在 block 中，我们可以使用 `MASConstraintMaker` 对象提供的各种方法描述视图之间的约束关系。

具体来说，当我们调用 `mas_makeConstraints` 方法时，Masonry 会为我们创建一个 `MASConstraintMaker` 对象，并将这个对象作为 block 的参数传递给我们。我们可以在 block 中使用这个 `MASConstraintMaker` 对象的各种方法描述视图之间的约束关系，例如：


```c
[view1 mas_makeConstraints:^(MASConstraintMaker *make) {     
make.left.top.equalTo(superview).offset(10);     
make.right.bottom.equalTo(superview).offset(-10); 
}];
```


在这个示例中，我们使用了 `make.left.top.equalTo(superview).offset(10)` 和 `make.right.bottom.equalTo(superview).offset(-10)` 这两个方法来描述视图之间的约束关系。这些方法的作用是将 `view1` 的左侧和顶部与 `superview` 的左侧和顶部对齐，并向右和向下偏移 10 个单位，同时将 `view1` 的右侧和底部与 `superview` 的右侧和底部对齐，并向左和向上偏移 10 个单位。

在 block 中，我们可以使用 `MASConstraintMaker` 对象的各种方法来描述视图之间的约束关系，例如设置视图的位置、大小、间距等。当 block 执行完毕后，Masonry 会根据我们在 block 中描述的约束关系来自动计算视图的位置和大小，并将这些约束关系应用到相应的视图上，从而实现自动布局。



## 下面是使用`make.width`点语法后的全部内部调用过程：
```c
// MASConstraintMaker
- (MASConstraint *)width {
    return [self addConstraintWithLayoutAttribute:NSLayoutAttributeWidth];
}

- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    return [self constraint:nil addConstraintWithLayoutAttribute:layoutAttribute];
}

- (MASConstraint *)constraint:(MASConstraint *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    // 根据 约束属性 和 视图 创建一个约束单元
    MASViewAttribute *viewAttribute = [[MASViewAttribute alloc] initWithView:self.view layoutAttribute:layoutAttribute];
    //创建约束，以约束单元作为约束的第一项
    MASViewConstraint *newConstraint = [[MASViewConstraint alloc] initWithFirstViewAttribute:viewAttribute];
    if ([constraint isKindOfClass:MASViewConstraint.class]) {
        //replace with composite constraint
        // 如果是在已有约束的基础上再创建的约束，则将它们转换成一个 组合约束，并将原约束替换成该组合约束。
        NSArray *children = @[constraint, newConstraint];
        MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
        compositeConstraint.delegate = self;
        // 这里会将原来 make.width 添加的约束 替换成一个 组合约束（宽度约束 + 高度约束）
        [self constraint:constraint shouldBeReplacedWithConstraint:compositeConstraint];
        // 返回组合约束
        return compositeConstraint;
    }
    if (!constraint) {// 如果不是在已有约束的基础上再创建约束，则添加约束至列表
        newConstraint.delegate = self;// 注意这一步，会对 make.top.left 这种情形产生关键影响
        [self.constraints addObject:newConstraint];
    }
    return newConstraint;
}
```
在第二次设置约束时（`.height`）会进入不同的流程。注意上面提到的`newConstraint.delegate`设置代理：
```c
//MAConstraint
- (MASConstraint *)height {
    return [self addConstraintWithLayoutAttribute:NSLayoutAttributeHeight];
}
//MSViewConstraint
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    NSAssert(!self.hasLayoutRelation, @"Attributes should be chained before defining the constraint relation");
	//delegate是MASConstraintMaker
    return [self.delegate constraint:self addConstraintWithLayoutAttribute:layoutAttribute];
}
// MASConstraintMaker
- (MASConstraint *)constraint:(MASConstraint *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {...}

```
下面看一下`.mas_equalTo(@100)`的流程。
```c
// MASConstraint
#define mas_equalTo(...)                 equalTo(MASBoxValue((__VA_ARGS__)))
- (MASConstraint * (^)(id))equalTo {
    return ^id(id attribute) {
        // attribute 可能是 @100 类似的值，也可能是 view.mas_width等这样的
        return self.equalToWithRelation(attribute, NSLayoutRelationEqual);
    };
}
- (MASConstraint * (^)(id))mas_equalTo {
    return ^id(id attribute) {
        return self.equalToWithRelation(attribute, NSLayoutRelationEqual);
    };
}
// MASViewConstraint
- (MASConstraint * (^)(id, NSLayoutRelation))equalToWithRelation {
    return ^id(id attribute, NSLayoutRelation relation) {
        if ([attribute isKindOfClass:NSArray.class]) {//是数组（有多个约束）
            NSAssert(!self.hasLayoutRelation, @"Redefinition of constraint relation");
            NSMutableArray *children = NSMutableArray.new;
            for (id attr in attribute) {
                MASViewConstraint *viewConstraint = [self copy];
                viewConstraint.layoutRelation = relation;
                viewConstraint.secondViewAttribute = attr;// 设置约束第二项
                [children addObject:viewConstraint];
            }
            MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
            compositeConstraint.delegate = self.delegate;
            [self.delegate constraint:self shouldBeReplacedWithConstraint:compositeConstraint];
            return compositeConstraint;
        } else {//单个约束
            NSAssert(!self.hasLayoutRelation || self.layoutRelation == relation && [attribute isKindOfClass:NSValue.class], @"Redefinition of constraint relation");
            self.layoutRelation = relation;
            self.secondViewAttribute = attribute;// 设置约束第二项
            return self;
        }
    };
}
// 设置约束第二项
- (void)setSecondViewAttribute:(id)secondViewAttribute {
	//判断类型
    if ([secondViewAttribute isKindOfClass:NSValue.class]) {
        [self setLayoutConstantWithValue:secondViewAttribute];
    } else if ([secondViewAttribute isKindOfClass:MAS_VIEW.class]) {
        _secondViewAttribute = [[MASViewAttribute alloc] initWithView:secondViewAttribute layoutAttribute:self.firstViewAttribute.layoutAttribute];
    } else if ([secondViewAttribute isKindOfClass:MASViewAttribute.class]) {
        _secondViewAttribute = secondViewAttribute;
    } else {
        NSAssert(NO, @"attempting to add unsupported attribute: %@", secondViewAttribute);
    }
}
// MASConstraint
- (void)setLayoutConstantWithValue:(NSValue *)value {
    if ([value isKindOfClass:NSNumber.class]) {
        self.offset = [(NSNumber *)value doubleValue];
    } else if (strcmp(value.objCType, @encode(CGPoint)) == 0) {
        CGPoint point;
        [value getValue:&point];
        self.centerOffset = point;
    } else if (strcmp(value.objCType, @encode(CGSize)) == 0) {
        CGSize size;
        [value getValue:&size];
        self.sizeOffset = size;
    } else if (strcmp(value.objCType, @encode(MASEdgeInsets)) == 0) {
        MASEdgeInsets insets;
        [value getValue:&insets];
        self.insets = insets;
    } else {
        NSAssert(NO, @"attempting to set layout constant with unsupported value: %@", value);
    }
}

// MASViewConstraint
- (void)setOffset:(CGFloat)offset {
    self.layoutConstant = offset;       // 设置约束常量
}
- (void)setSizeOffset:(CGSize)sizeOffset {
    NSLayoutAttribute layoutAttribute = self.firstViewAttribute.layoutAttribute;
    switch (layoutAttribute) {
        case NSLayoutAttributeWidth:
            self.layoutConstant = sizeOffset.width;
            break;
        case NSLayoutAttributeHeight:
            self.layoutConstant = sizeOffset.height;
            break;
        default:
            break;
    }
}
- (void)setCenterOffset:(CGPoint)centerOffset {
    NSLayoutAttribute layoutAttribute = self.firstViewAttribute.layoutAttribute;
    switch (layoutAttribute) {
        case NSLayoutAttributeCenterX:
            self.layoutConstant = centerOffset.x;
            break;
        case NSLayoutAttributeCenterY:
            self.layoutConstant = centerOffset.y;
            break;
        default:
            break;
    }
}
- (void)setInsets:(MASEdgeInsets)insets {
    NSLayoutAttribute layoutAttribute = self.firstViewAttribute.layoutAttribute;
    switch (layoutAttribute) {
        case NSLayoutAttributeLeft:
        case NSLayoutAttributeLeading:
            self.layoutConstant = insets.left;
            break;
        case NSLayoutAttributeTop:
            self.layoutConstant = insets.top;
            break;
        case NSLayoutAttributeBottom:
            self.layoutConstant = -insets.bottom;
            break;
        case NSLayoutAttributeRight:
        case NSLayoutAttributeTrailing:
            self.layoutConstant = -insets.right;
            break;
        default:
            break;
    }
}
```
另外，后面的 `offset` 方法做了一步额外的操作：
```c
// MASConstraint
- (MASConstraint * (^)(CGFloat))offset {
    return ^id(CGFloat offset){
        self.offset = offset;
        return self;
    };
}
- (void)setOffset:(CGFloat)offset {
    self.layoutConstant = offset;
}

```

```c
- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *make))block {

    NSMutableArray *constraints = [NSMutableArray array];

    for (MAS_VIEW *view in self) {

        NSAssert([view isKindOfClass:[MAS_VIEW class]], @"All objects in the array must be views");

        [constraints addObjectsFromArray:[view mas_makeConstraints:block]];

    }

    return constraints;

}
```
该方法用于为一个包含多个视图的数组设置约束。具体地，这个方法会遍历数组中的每个视图，并对每个视图调用其自身的 `mas_makeConstraints` 方法，将传入的 `block` 块作为参数传递给它。

`block` 块参数中的 `MASConstraintMaker` 对象包含了各种用于描述视图约束的方法。在调用 `mas_makeConstraints` 方法时，传入的 `block` 块中可以使用这些方法来描述视图之间的相对位置、尺寸等信息。`mas_makeConstraints` 方法会根据这些信息计算出合适的约束，并将其添加到相应的视图上。

在该方法中，为了遍历数组中的每个视图并添加约束，我们使用了 `for-in` 循环。对于每个视图，我们首先使用 `NSAssert` 函数来断言其是否属于 `MAS_VIEW` 类型，如果不是则会抛出异常，以保证该数组只包含视图对象。接着，我们将视图的 `mas_makeConstraints` 方法返回的约束添加到 `constraints` 数组中。最后，该方法返回 `constraints` 数组，其中包含了为数组中的每个视图设置的约束。