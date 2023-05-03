# new的源码
**+ new**  **{**
**id newObject = (*_alloc)((Class)self, 0);**
**Class metaClass = self->isa;**
**if (class_getVersion(metaClass) > 1)** {
**return [newObject init];**
} **else** {
**return newObject;**
}
**}**

# alloc init源码

**+ alloc**

**{**

**return (*_zoneAlloc)((Class)self, 0, malloc_default_zone());**

**}**

**- init**

**{**

**return self;**

**}**

**通过源码中我们发现，[className new]基本等同于[[className alloc] init]；**

**区别只在于alloc分配内存的时候使用了zone.**

**这个zone是个什么东西呢？**

**它是给对象分配内存的时候，把关联的对象分配到一个相邻的内存区域内，以便于调用时消耗很少的代价，提升了程序处理速度；**
**3.而为什么不推荐使用new？**

**不知大家发现了没有：如果使用new的话，初始化方法被固定死只能调用init.**

**而你想调用initXXX怎么办？没门儿！据说最初的设计是完全借鉴Smalltalk语法来的。**

**传说那个时候已经有allocFromZone:这个方法，**

**但是这个方法需要传个参数id myCompanion = [  [ TheClass allocFromZone:[self zone] ] init ];**

**这个方法像下面这样：**

**+ allocFromZone:(void *) z**

**{**

**return (*_zoneAlloc)((Class)self, 0, z);**

**}**

**//后来简化为下面这个：**

**+ alloc**

**{**

**return (*_zoneAlloc)((Class)self, 0, malloc_default_zone());**

**}**

**但是，出现个问题：这个方法只是给对象分配了内存，并没有初始化实例变量。**

**是不是又回到new那样的处理方式：在方法内部隐式调用init方法呢？**

**后来发现“显示调用总比隐式调用要好”，所以后来就把两个方法分开了。**

**概括来说，new和alloc/init在功能上几乎是一致的，分配内存并完成初始化。**

**差别在于，采用new的方式只能采用默认的init方法完成初始化，**

**采用alloc的方式可以用其他定制的初始化方法。**