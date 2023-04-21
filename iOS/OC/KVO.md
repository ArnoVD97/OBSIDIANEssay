# Key-value observing

Key-value observing is a mechanism that enables an object to be notified directly when a property of another object changes. Key-value observing (or KVO) can be an important factor in the cohesiveness of an application. It is a mode of communication between objects in applications designed in conformance with the Model-View-Controller design pattern. For example, you can use it to synchronize the state of model objects with objects in the view and controller layers. Typically, controller objects observe model objects, and views observe controller objects or model objects. 

**Note:** Although the classes of the UIKit framework generally do not support KVO, you can still implement it in the custom objects of your application, including custom views.

![Art/kvo_2x.png](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/Art/kvo_2x.png)

With KVO, one object can observe any properties of another object, including simple attributes, to-one relationships, and to-many relationships. An object can find out what the current and prior values of a property are. Observers of to-many relationships are informed not only about the type of change made, but are told which objects are involved in the change. 

As a notification mechanism, key-value observing is similar to the mechanism provided by the `[NSNotification](https://developer.apple.com/documentation/foundation/nsnotification)`and `[NSNotificationCenter](https://developer.apple.com/library/archive/documentation/LegacyTechnologies/WebObjects/WebObjects_3.5/Reference/Frameworks/ObjC/Foundation/Classes/NSNotificationCenter/Description.html#//apple_ref/occ/cl/NSNotificationCenter)` classes, but there are significant differences, too. Instead of a central object that broadcasts notifications to all objects that have registered as observers, KVO notifications go directly to observing objects when changes in property values occur.

## Implementing KVO

The root class, `[NSObject](https://developer.apple.com/library/archive/documentation/LegacyTechnologies/WebObjects/WebObjects_3.5/Reference/Frameworks/ObjC/Foundation/Classes/NSObject/Description.html#//apple_ref/occ/cl/NSObject)`, provides a base implementation of key-value observing that you should rarely need to override. Thus all Cocoa objects are inherently capable of key-value observing. To receive KVO notifications for a property, you must do the following things:

-   You must ensure that the observed class is _key-value observing compliant_ for the property that you want to observe.
    
    KVO compliance requires the class of the observed object to also be KVC compliant and to either allow automatic observer notifications for the property or implement manual key-value observing for the property.
    
-   Add an observer of the object whose value can change. You do this by calling `[addObserver:forKeyPath:options:context:](https://developer.apple.com/documentation/objectivec/nsobject/1412787-addobserver)`. The observer is just another object in your application.
    
-   In the observer object, implement the method `[observeValueForKeyPath:ofObject:change:context:](https://developer.apple.com/documentation/objectivec/nsobject/1416553-observevalueforkeypath)`. This method is called when the value of the observed object’s property changes.
    

## KVO Is an Integral Part of Bindings (OS X)

Cocoa bindings is an OS X technology that allows you to keep values in the model and view layers of your application synchronized without having to write a lot of “glue code.” Through an Interface Builder inspector, you can establish a mediated connection between a view’s property and a piece of data, “binding” them such that a change in one is reflected in the other. KVO, along with key-value coding and key-value binding, are technologies that are instrumental to Cocoa bindings.

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