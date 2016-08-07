---
layout: default
title: Object culling (C#)
category: welkin_road
---
<div class="content-inner">
    <div class="container">
        <h1>Object culling</h1>

        <h3>Project:</h3>
        <p>
            <a href="/projects/welkin_road.html">Welkin Road</a>
        </p>

        <h3>Programming language:</h3>
        <p>
        	C#
        </p>

        <h3>Engine:</h3>
        <p>
            Untiy 3D
        </p>

        <h3>Description</h3>
        <p>
            Sometimes objects or parts of objects need to be activated/deactivated (culled) based on some criterium such as distance. The abstract classes ObjectCuller and CulledObject allow any component of any object to be activated and deactivated based on a chosen criterum. They also allow the user to balance performance cost against frequency of updates by controlling the maximum number of objects that will be culled per frame. ObjectCuller represents a blueprint for handling CulledObjects. It is up to the concrete classes to choose which components of objects will be affected and in what way. An example of such concrete classes are ParticleSystemCuller and ParticleSystemCulledObject.
      	</p>

      	<h3>ObjectCuller.cs</h3>
{% highlight cs %}
using UnityEngine;

public abstract class ObjectCuller : MonoBehaviour
{
    public GameObject objects;
    public float objectsToCullPerFrame = 1;

    // concrete objects that inherit from CulledObject
    private CulledObject[] culledObjects;
    private int culledObjectIndex;

    private void Start()
    {
        culledObjects = GetCulledObjects();
        if (culledObjects.Length <= 0)
        {
            enabled = false;
        }
    }

    private void Update()
    {
        for (int i = 0; i < objectsToCullPerFrame; ++i)
        {
            culledObjects[culledObjectIndex].Cull();

            // update index
            ++culledObjectIndex;
            culledObjectIndex %= culledObjects.Length;
        }
    }

    // gets the concrete objects
    public abstract CulledObject[] GetCulledObjects();

}
{% endhighlight %}

    <h3>CulledObject.cs</h3>
{% highlight cs %}
public abstract class CulledObject
{
    public bool Active { get { return active; } }

    private bool active = true;

    // to cull means to activate/deactivate objects
    public void Cull()
    {
        if (!active && !IsToBeCulled())
        {
            active = true;
            Activate();
        }
        else if (active && IsToBeCulled())
        {
            active = false;
            Deactivate();
        }
    }

    // culling criterium
    public abstract bool IsToBeCulled();

    protected abstract void Activate();
    protected abstract void Deactivate();
}
{% endhighlight %}

    <h3>ParticleSystemCuller.cs</h3>
{% highlight cs %}
using UnityEngine;
using System.Collections.Generic;

public class ParticleSystemCuller : ObjectCuller
{
    public float drawDistance;

    // limit search depths
    public int searchDepth = 1;

    public override CulledObject[] GetCulledObjects()
    {
        return GetCulledObjectsRecursive(objects.transform, searchDepth);
    }

    // create a ParticleSystemCulledObject for every particle system
    private CulledObject[] GetCulledObjectsRecursive(Transform obj, int depth)
    {
        --depth;
        List<CulledObject> culledObjs = new List<CulledObject>();
        foreach (Transform child in obj.transform)
        {
            ParticleSystem ps = child.GetComponent<ParticleSystem>();
            if (ps != null)
            {
                ParticleSystemCulledObject culledObject = new ParticleSystemCulledObject();
                culledObject.player = GameObject.FindGameObjectWithTag(Tags.player);
                culledObject.obj = child.gameObject;
                culledObject.drawDistance = drawDistance;
                culledObject.particleSystem = ps;
                if (culledObject.IsValid())
                {
                    culledObjs.Add(culledObject);
                }
            }
            else if (depth >= 1)
            {
                culledObjs.AddRange(GetCulledObjectsRecursive(child, depth));
            }
        }
        return culledObjs.ToArray();
    }
}
{% endhighlight %}

    <h3>ParticleSystemCulledObject.cs</h3>
{% highlight cs %}
using UnityEngine;

public class ParticleSystemCulledObject : CulledObject
{
    public GameObject player;
    public GameObject obj;
    public float drawDistance;
    public ParticleSystem particleSystem;

    public bool IsValid()
    {
        return player != null && obj != null;
    }

    public override bool IsToBeCulled()
    {
        // use distance as culling criterium
        return Vector3.Distance(player.transform.position, obj.transform.position) > drawDistance;
    }

    protected override void Activate()
    {
        if (particleSystem != null)
        {
            particleSystem.Play();
        }
    }

    protected override void Deactivate()
    {
        if (particleSystem != null)
        {
            particleSystem.Stop();
        }
    }
}
{% endhighlight %}

    </div>
</div>