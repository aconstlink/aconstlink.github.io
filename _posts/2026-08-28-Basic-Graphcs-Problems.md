---
layout: post
title:  "Basic Graphics Problems"
categories: graphics demos
---

I am on to create a graphics demo using my engine [motor](https://github.com/aconstlink/motor). For the demo, of course, I also create tools and a early preview looks like this:
![early view of demo](/assets/images/img1.png)

The image shows the current state of the tool. On the left is the tool window where you can control the demo, on the top right is the debug window, where you see a very basic, unlit and no-shadow view of the scene. On the bottom right you see the production window which shows the full blown demo.

The application has a scene manager where a scene can be loaded and unloaded based on the scenes' requirments. Also all graphics objects are initialized and relesed if the scene is loaded and unloaded. The load and unload is based on a time period a scene is valid. 

The application allows also to open and close the production window in order to save some compuation power while the demo is being developed. 

# Dynamic Object Initialzation Problem

A big problem developing this demo revealed that the engine did not support some dynamics. A graphics object is supposed to be configured first before it can be used. Sounds obvious but in reality it required the user to do a configure for every change. Unfortunately, using dynamic geometry or variable sets was not possible without a reconfigure. A reconfigure on the otherhand is very costly with a shader recompilation required.

So the engine needed a dynamic geometry and variable set change api.

## What is required?

It is required to render multiple objects with the same shader. Multiple objects means multiple geometries and each with its own variable set, so that geometry can be placed in the scene. Lets just see how those parts are linked together:

![Relation](/assets/images/msl_renderobj_relation01.png)

**msl object:** A msl object represents a motor shading language object. The user can load a msl shader and get all the features it supports.

**render object:** A render object represents a api shader. Internlly in the backends, the msl shader is translated to an api shader. For an OpenGL 4 backend that would be a GLSL shader and for a Direct3D 11 backend, that would be a HLSL 5 shader. 

**Geometry Link:** A geometry link allows the user to link a previously configured geometry object to the shader for linkage. That means an id is generated for that geometry link which can be used for rendering. 

**Variable Set:** A Variable Set allows the user to specify shader variables which can be changed during application run-time. So any shader variable can be changed using this interface.

The problem now was that if a msl object was desired to be used for multiple objects, that shader needed to be configured multiple times. There was no possibility for the user to reuse a shader. Yes thats true. Originally, the design wasn't to render 3d scenes dynamically. Now it is and the engine needed support that feature. For example, a depth pass shader(pre-z) renders all the objects currently in the view region into the depth buffer. All those objects are not initialzied with any depth pass shader. The depth pass shader is provided by the engine and just renders the geometric objects' depth values into the depth buffer. So the engine needs to inject objects into msl objects during run-time by linking geometry and supply a variable set.

Because I implemented all the backends one or two year back, it took some time to change the various places in the code to allow for dynamic objects in the msl/render objects. 

Now it is possible to dynamically link a new geometry or a new variable set and the backend will recognize the new data and react accordingly, so the new object ids can be used for rendering.

# Why is this important?

Using just one msl for shadow or depth passes is very efficient. Every time a new object from the scene comes into viewable region, its geometry needs to be attached to the paricular depth pass msl shader along with its variable set so the geometry can be propery positioned in the view space for rendering into the depth buffer. There are more examples like default shaders when you do just want to see something on the screen for testing for example. 

Anyways, I can not go into further detail because it is very complex and this post would be very long nobody would read anyways. The puzzly will form into a broader picture later down the blog path. 

It is also important because it just shows that the engine can initialze and release cycle through graphics objects without crash and with correctly display. It is very important to not leave any leaks after cleanup. 

# Conclusion

At the moment of writing, only the D3D11 backend was modified to support that feature. This feature also revealed more inefficiencies in the backend and lead me to fix some leaks too. 

Another big time eater was the missed assignment of default values. msl supports assignment of default values to shader variables. Those defaults are set in the backend when the shader is compiled. Because of the old behaviour, defaults where assigned after the shader was complied, but with dynamic varialbe sets, the default values actually need to be assigned when the new variable set realized in the backend even if there is no shader recompilation.

What really helped was to have the [test suites](https://github.com/aconstlink/motor_suites) where I implemented and tested the new feature. In there it is possible to go really low on engine level and test things out.

During implementation of the new feature, I realized there need to be some cleanup and refactoring in the engine. I even forgot about that I removed the frame based async rendering. I remember I did it because shader variables could not easily be sent on the proper time point before rendering. This feature needs to be reimplemented. With this flaw also comes shader variable management. Those need to be packed more densly for GPU transmission.

Also it took too long to implement the dynamic geometry and varialbe set feature because of me interatively implementing the engine. The code "grows" and many features should not be like they are. 

Rendering also became too lazy. Too many things are checked with a render call. That needs to be refactored too. The rendering api needs to be more finely grained so things like linking geometry or adding new variable sets can be done separatly before anythings is rendered or when an object is prepared for rendering without an expensive configure call.

# Quick recall

## The case that was not working

The original problem was that it was not possible to inject msl shaders into a scene graph. The image on the very top just shows a scene exported from Blender using gltf. The engine imports that gltf scene and creates a scene graph. In that scene graph, the renderalbe nodes contain a default msl shader that can be used as was defined by the gltf file. But chaning the rendering pipeline to have a pre-z pass, that pre-z shader needs to be injected into the scene graphs' renderable nodes so a pre-z pass render visitor could travel the scene graph and send the corresponding objects to render. Injecting the depth pass shader was not possible before, so now it is and the geometry contained in the renderable node is just attached to the msl depth pass shader so a pre-z pass can be deployed across the scene.

So without this change, the user would have been required to load and compile the depth pass shader per renderable object. That is insane because msl compilation takes some time and shader compilation in general also takes some time. So that was not possible, the new feature needed to be done. Also the user needed to manage all those shader too.