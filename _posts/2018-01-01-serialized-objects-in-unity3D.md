---
layout: post
title: Serialized Objects inside Unity3D
description: A quick look into serialization and serialized objects inside Unity-Technologies engine Unity3D
keywords: Unity3D, Serialized-Objects, C#
tags: [Unity3D, C#, Serialization, Documentation]
comments: false
---

["Serialization is the automatic process of transforming data structures or object states into a format that Unity can store and reconstruct later."](https://docs.unity3d.com/Manual/script-Serialization.html)

Serialization is a basic feature of [.NET](https://docs.microsoft.com/en-us/dotnet/standard/serialization/) applications and therefore at the very core of Unity3D (Still there is a difference between .NET serialization and Unity's serialization system). Everything in Unity3D gets serialized at one point or another. Every Object, Texture, Scene you Create, Instantiate, Copy, Delete, Edit gets serialized so it's current state can be saved and send to the Memory, Storage, Database, or else. 

So serialization is performed to convert the state of an object into a form (Binary or XML) that can be saved or transported over the Network or over to different applications. 

#### Here are some examples of serialized data in Unity3D:

+ **Prefabs:** Internally, a perfab is the serialized data stream of one (or more) game objects as well as the components of it. The instance of a prefab is a list of modifications made to the prefab. It shows how the serialized data has to be modified to get the desired result.
<br><br>
+ **Unity3D's Inspector window:** The built-in inspector window inside of Unity3D's Editor also serializes objects before it is able to display them to you (It doesn't talk directly to the C#-API). So whenever it retrieves basic data of different GameObjects like for example Transform.position or else, it will serialize the object, load the serialized data and then display it to you. This can also be explained by the [SerializeField] attribute which lets you display private fields inside the inspector.
<br><br>
+ **Instantiation:** Calling Instantiate(); on a GameObject or prefab will *serialize* the object, *create* the new object and then *deserialize* the serialized data so the necessary modifications can be made to new object. 
<br><br>
+ **Saving & Loading:** Saving or loading *.unity* scene files will use [YAML](https://www.sitepoint.com/choosing-right-serialization-format/) to serialize the data inside the scene. Now everytime you load your scene, Unity3D is going to completely rebuild your scene from scratch by reading the YAML file.
<br><br>
+ **Storing data:** which is stored in your scripts

**TODO:** Insert picture of serialized data

> Serialized Data from a typical Unity3D scene file: "perlin_vectorFields.unity" 

#### When exactly does Unity3D serialize Objects?

Unity3D's full inner workings still remain a mistery, but this is taken from [Lucas Meijer](https://blogs.unity3d.com/author/lucas/) explaining it in a [post](https://blogs.unity3d.com/2014/06/24/serialization-in-unity/) on the official Unity3D-Blog. 

**Unity3D serializes the following type of objects:**

+ public or fields with the [SerializeField] attribute
+ Non static Fields
+ Non const Fields
+ Non readonly Fields

**The following can be serialized:**

+ Custom non abstract classes with [[Serializable]](https://docs.unity3d.com/ScriptReference/Serializable.html) attribute
+ Custom structs with [[Serializable]](https://docs.unity3d.com/ScriptReference/Serializable.html) attribute. (new in Unity4.5)
+ References to objects that derive from UntiyEngine.Object
+ Primitive data types (int, float, double, bool, string, etc)
+ Array of a fieldtype we can serialize
+ List<T> of a fieldtype we can serialize

### What do I need to look out for when working with scripts that will be serialized?

#### 1.Unity3D does not support serialization of null fields

Whenever the serializer encounters a field that is null, it will automatically instantiate a new object of the same type and then serialize that. In this case however the serialized object will also be null and therefore the serializer will be stuck in an endless loop. **To counter the loop Unity3D has a serialization limit of seven levels.**

#### 2.Custom class'es behave the same way like struct's

```csharp 
[Serializable]
class Animal {
    public string name;
}

class World : MonoBehaviour {
    public Animal[] animals;
}

  ```
> If the World class has an Animal array with three members, then you will find 3 objects inside the serialization-stream. 

#### 3.Custom polymorphic classes are not supported

If you are trying to serialize polymorphic class'es that do not inherit from ScriptableObject or MonoBehaviour it won't work, because Unity3D's serialization system doesn't support inheritance for custom serializable class'es. Here is a example:


```csharp

[System.Serializable]
public class Base : MonoBehaviour {
    public Child myChild;

}

[System.Serializable]
public class Child {
    public float age;
}

```

Will produce the following result:

**TODO:** Insert picture of inspector without polymorphism

> You will get a dropdown menu displaying the elements of the Child-object. You can even edit them.
Changing the Child class so that it derives from Base will produce the following:

```csharp
[System.Serializable]
public class Child : Base
```

**TODO:** Insert picture of inspector with polymorphism

> The inspector is not able the retrieve the value anymore as the Unity3D's serialization system can't resolve the polymorphic classes used in this example. The example was taken from the following StackExchange post: ["Unity Inspector Not Showing Serialized Variables Of Child"](https://gamedev.stackexchange.com/questions/98994/unity-inspector-not-showing-serialized-variables-of-child/99000#99000). It took it, because it was a pretty good example.

<div class="divider"></div>

# Optimizing serialized Content

Optimizing your game's content is one of the last steps to make when you are building your game, more specifally: It's what you do before you finally release your game. So bascially after you made a game-concept, build the most basic element of it and included all the additional ones you will look at your game and wonder: "How do I make it faster?".

> "Make your game fun, make your game work... Then make it fast."
    - Ian Dundore

Well for that we first have to take a look at all the elements that are affect by Unity3D's serialization system and take that into consideration everytime we work with them. We do this, because serialization is helpful in a lot of ways,but it can also be very destructive. This is due to the fact that sometimes we forget about all the elements that will be serialized and therefore create unnecessary operations that will only slow down our game. So the goal is the develop kind of a sixth sense that tells us when we have to take care about our serializations in the scene. <br>

**Here are some basic things in Unity3D to look out for:**

+ Instantiated objects
+ Transforms
+ UI-Elements
+ Animation Rig's (Bones)

When used the wrong way, these elements of Unity3D can lead to severe problems in our final build. To give you some examples I took these examples from ["Ian Dundore's Unity Talk about Content-Optimization"](https://www.youtube.com/watch?v=n-oZa4Fb12U).

<div class="divider"></div>

#### Test Example: 10.000 Objects

**Create: 10.000 Objects manually**
Create about 10.000 Objects manually by instantiating a base object and then by adding all the necessary components. 

```csharp
void CreateObjects() {

    GameObject root = new GameObject();
    Transform parent = root.Transform;

    for (int i = 0; i < 10000; i++) {

        GameObject newObj = new GameObject();
        newObj.AddComponent<TestMonoBehaviour>();
        newObj.transform.parent = parent;

    }

}
```

**Load: 10.000 Objects from Prefab**
Instead of copying the data and recreating the hierarchy you can also just create a Prefab for it to load from it later on when you need it in your game.

```csharp
void LoadObjects() {

    GameObject prefab = Resources.Load("ObjPrefab");
    GameObject.Instantiate(prefab);

}
```

**Clone: 10.000 Objects from a single Template**
When creating multiple objects that all share specific preferences or are completely identical, then you can also create a template prefab, load it and then clone it.

```csharp
void CloneObjects() {

    GameObject root = new GameObject();
    Transform parent = root.transform;

    GameObject newObj = null;
    for (int i = 0; i < 10000; i++) {

        if (newObj =0 null) {

            newObj = new GameObject();
            newObj.AddComponent<TestMonoBehaviour>();
            newObj.transform.parent = parent;

        } else {
            newObj = GameObject.Instantiate(newObj, parent);
        }

    }

}
```

> Fastest method


#### **Test Results**

|                 | Create  | Load    | Clone   |
| --------------- | -------:| -------:| -------:|
| Editor          | 310 ms  | *94 ms* | 125 ms  | 
| Win64 (SSD)     | 6.5 ms  | 23.5 ms | *4 ms*  |
| iOS (Air 2)     | 583 ms  | 1366 ms | *188 ms*|

> Timings (Platform dependent)

There are some things to consider when looking at this chart: The first thing is that inside the Unity-Editor you will most likely get faster Loading/Caching times because Unity3D does that in the background while you work. If you start you Unity3D Game In-Editor for the first time it will load all of the objects and keep them loaded until you close Unity. So thats why Loading is so fast inside the Unity-Editor, because all of the objects are preloaded in the scene.

When it comes to Windows then Loading is only fast if you have dedicated Hardware. The Loading time on Windows/Linux/Mac/Android will depend on the storage-location of your game: SSD/HDD/"Slow SD-Card"/... Slow Storage is something you need to take in consideration -> Asynchronous Loading on a worker-Thread will help on this

So why is Create() so slow - Using AddComponent will force the Unity3D engine to look up the scripting class it must attach. For a short script with no Awake() call this can take about 403ms to do. If you have a Awake() call in your script you are trying to load then the process-time of that function will also be added to the Total Time spend on creating the objects: 10.000 Objects + 10.000 Awake() Calls... The Total Time can be split into the following:

$$TotalTime = T_addComponent + T_spawnObject + T_reparenting + T_awakeCalls * n$$

**So why is cloning the fastest way to create multiple objects of the same type?**

|    | Total | Spawn GameObjects | Add components | Reparenting | Remainder |
| -- | ---------- | ----------------- | -------------- | ----------- | --------- |
| Clone | 188 ms | 175 / 112 ms | 0 / 63 ms | 0 / 2 ms | 13 ms |

**Steps to improve load times and how to avoid unnecessary ones:**

Better Game Content Structure - Example:

If you do it like its then you will end up with multiple, identical enemies in your scene that all share the same script and in at least some ways the same preferences. To improve this you can transfer the settings-data for the script onto a ScriptableObject - One File, One Asset, 10.000 references to it

Dont put Object-Pools in another Transform -> Dont tidy up your project with transforms -> This is going to create more & more serialized objects that only slow down your game. -> Keep everything dirty in your release builds

<div class="divider"></div>

### Animator performance & Transforms

#### Optimize Game Object - Checkbox

When you add character models with animations and a rig then Unity3D will create a Transform for every bone the character/model has -> this will create a lot of data that needs to be loaded and will slow down you game. To counter this go into the model-prefab -> to the Rig Tab and select "Optimize Game Object" -> This will only create a Transform for the Root-Bone. 

It will also reorder the mecanim animation data to make it more mulithreadable... If there are some transforms on your model you need for game-development e.g.: Anchor-Bone for weapon then you can expose those with the "Extra Transforms to Expose" field right beneath the checkbox. There you will be able to select the bones you want to expose for your needs.

|             | iPad Air 2 | Macbook Pro 2015 | PC 2012 |
|:----------- |:----------:|:----------------:|:-------:|
| Unoptimized | 18.14 ms   | 3.62 ms          | 3.47 ms |
| Two Bones   | 10.07 ms   | 1.66 ms          | 1.31 ms |
| Optimized   | 9.72 ms    | 1.49 ms          | 1.27 ms |

<div class="divider"></div>

### Unity3D User Interface Elements
#### *Creating smooth Interfaces*

Canvas - Finds the best way to draw the components of your UI to the screen - batches these and sends them to the graphics card. If any one thing in the UI changes - any sprite/text/image - then the UI needs to recalculate all of its draw calls. It will take all of the UI objects and sorts them in the order like they are drawn by the graphics card - so the Canvas will be sorted by depth... 

"A canvas rebuilds if any Drawable components changes" 

The Problem: A lot of UI's consist of many different transforms/objects/... sorting these is a huge task for the canvas which is scaling logarithmically per object. Rebuilding these everytime a object changes is *very* expensive 

"All Unity UI is drawn as Transparent geometry" -> All UI Elements have an Alpha value ... That means that every pixel in every quad will be sampled by the graphics card and therefore when you have things like fullscreen-textures laid a top of each other then it will end up sampling all of these even through they are at the back e.g:

4 Fullscreen textures on top of each other -> every texture is going to be sampled -> quads are not culled

#### What to do against that?

**Split up your UI into multiple Canvas'es**

Seperate your UI into different sections and use a seperate Cavans for each of them - it is possible to nest a cavans within another canvas. This will make sure that everytime a object changes only the specific part gets redrawn & reordered so it can be send to the graphics card significantly faster. 

Those nested cavases isolate their children from rebuilds. So seperate you UI into different parts by their update cycles. Things that get updated every frame are in a seperate canvas than things that are static and dont ever update inside the game:

> Dynamic Canvas - *Updates every frame*<br>
> Background Canvas - *Updates once*<br>
> Static Canvas - *Updates once and then never again*<br>

**Combine Objets/Sprites/Text Elements**

Merge all of the objects you can in your UI. E.g.: If you create a table then dont create a TextObject for each column/line - create one text object and span it across the table