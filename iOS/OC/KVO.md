# Key-value observing

Key-value observing is a mechanism that enables an object to be notified directly when a property of another object changes. Key-value observing (or KVO) can be an important factor in the cohesiveness of an application. It is a mode of communication between objects in applications designed in conformance with the Model-View-Controller design pattern. For example, you can use it to synchronize the state of model objects with objects in the view and controller layers. Typically, controller objects observe model objects, and views observe controller objects or model objects. 

**Note:** Although the classes of the UIKit framework generally do not support KVO, you can still implement it in the custom objects of your application, including custom views.

键值观察是一种机制，当另一个对象的属性发生变化时，可以直接通知另一个对象。键值观察(KVO)是影响应用程序内聚性的一个重要因素。它是一种应用程序中对象之间的通信模式，设计遵循MVC设计模式。例如，您可以使用它将模型对象的状态与视图和控制器层中的对象同步。通常，控制器对象观察模型对象，而视图观察控制器对象或模型对象。

**注意:**尽管UIKit框架的类通常不支持KVO，但您仍然可以在应用程序的自定义对象中实现它，包括自定义视图。

![Art/kvo_2x.png](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/Art/kvo_2x.png)

With KVO, one object can observe any properties of another object, including simple attributes, to-one relationships, and to-many relationships. An object can find out what the current and prior values of a property are. Observers of to-many relationships are informed not only about the type of change made, but are told which objects are involved in the change. 

As a notification mechanism, key-value observing is similar to the mechanism provided by the [NSNotification](https://developer.apple.com/documentation/foundation/nsnotification) and  [NSNotificationCenter](https://developer.apple.com/library/archive/documentation/LegacyTechnologies/WebObjects/WebObjects_3.5/Reference/Frameworks/ObjC/Foundation/Classes/NSNotificationCenter/Description.html#//apple_ref/occ/cl/NSNotificationCenter)classes, but there are significant differences, too. Instead of a central object that broadcasts notifications to all objects that have registered as observers, KVO notifications go directly to observing objects when changes in property values occur.

使用KVO，一个对象可以观察另一个对象的任何属性，包括简单属性、一对关系和多关系。对象可以找出属性的当前值和之前值。对多关系的观察者不仅被告知所做更改的类型，而且还被告知哪些对象参与了更改。

作为一种通知机制，键值观察类似于[NSNotification](https://developer.apple.com/documentation/foundation/nsnotification)和[NSNotificationCenter](https://developer.apple.com/library/archive/documentation/LegacyTechnologies/WebObjects/WebObjects_3.5/Reference/Frameworks/ObjC/Foundation/Classes/NSNotificationCenter/Description.html#//apple_ref/occ/cl/NSNotificationCenter)类提供的机制，但也有显著的差异。当属性值发生变化时，KVO通知直接发送给观察对象，而不是一个向所有注册为观察者的对象广播通知的中心对象。


## 实现KVO

The root class, [NSObject](https://developer.apple.com/library/archive/documentation/LegacyTechnologies/WebObjects/WebObjects_3.5/Reference/Frameworks/ObjC/Foundation/Classes/NSObject/Description.html#//apple_ref/occ/cl/NSObject), provides a base implementation of key-value observing that you should rarely need to override. Thus all Cocoa objects are inherently capable of key-value observing. To receive KVO notifications for a property, you must do the following things:

-   You must ensure that the observed class is _key-value observing compliant_ for the property that you want to observe.
    
    KVO compliance requires the class of the observed object to also be KVC compliant and to either allow automatic observer notifications for the property or implement manual key-value observing for the property.
    
-   Add an observer of the object whose value can change. You do this by calling [addObserver:forKeyPath:options:context:](https://developer.apple.com/documentation/objectivec/nsobject/1412787-addobserver). The observer is just another object in your application.
    
-   In the observer object, implement the method [observeValueForKeyPath:ofObject:change:context:](https://developer.apple.com/documentation/objectivec/nsobject/1416553-observevalueforkeypath). This method is called when the value of the observed object’s property changes.
    

根类[NSObject]提供了一个基本的键值观察实现，你应该很少需要覆盖它。因此，所有Cocoa对象都具有键值观察能力。为了接收属性的KVO通知，你必须做以下事情:

-你必须确保被观察的类对你想要观察的属性是* *键值兼容* *的。

符合KVO规范要求被观察对象的类也符合KVC规范，允许属性的观察者自动通知，或者实现属性的键值手动观察。

—添加可以改变值的对象的观察者。你可以通过调用[addObserver:forKeyPath:options:context:]来做到这一点，观察者只是应用程序中的另一个对象。

—在观察者对象中，实现[observeValueForKeyPath:ofObject:change:context:]方法。当被观察对象的属性值发生变化时，将调用此方法。
## KVO是绑定的组成部分(OS X)

Cocoa bindings is an OS X technology that allows you to keep values in the model and view layers of your application synchronized without having to write a lot of “glue code.” Through an Interface Builder inspector, you can establish a mediated connection between a view’s property and a piece of data, “binding” them such that a change in one is reflected in the other. KVO, along with key-value coding and key-value binding, are technologies that are instrumental to Cocoa bindings.

Cocoa绑定是一种OS X技术，它允许你同步应用程序的模型层和视图层中的值，而无需编写大量的“粘合代码”。通过Interface Builder检查器，您可以在视图的属性和数据之间建立一个中介连接，“绑定”它们，以便其中一个的更改反映到另一个中。KVO、键值编码和键值绑定是Cocoa绑定的关键技术。

### Prerequisite Articles

-   [Key-value coding](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/KeyValueCoding.html#//apple_ref/doc/uid/TP40008195-CH25-SW1)

### Related Articles

-   [Model-View-Controller](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/MVC.html#//apple_ref/doc/uid/TP40008195-CH32-SW1)
-   [Dynamic binding](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/DynamicBinding.html#//apple_ref/doc/uid/TP40008195-CH15-SW1)

### Definitive Discussion

_[Key-Value Observing Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html#//apple_ref/doc/uid/10000177i)_

### Sample Code Projects

-   [SourceView: Using NSOutlineView with NSTreeController](https://developer.apple.com/library/archive/samplecode/SourceView/Introduction/Intro.html#//apple_ref/doc/uid/DTS10004441)
-   [BindingsJoystick](https://developer.apple.com/library/archive/samplecode/BindingsJoystick/Introduction/Intro.html#//apple_ref/doc/uid/DTS10003684)