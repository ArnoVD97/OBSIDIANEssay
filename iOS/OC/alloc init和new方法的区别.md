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
