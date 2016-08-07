---
layout: default
title: Shooting star system (C#)
category: welkin_road
---
<div class="content-inner">
    <div class="container">
        <h1>Shooting star system</h1>

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
            Creates a shooting star effect at randomized time intervals, randomized locations on a dome around the player and randomized downward facing directions. ShootingStar handles the effects while ShootingStarManager handles the spawning. An object pool is used for the spawning of shooting stars so that the same objects are being reused, avoiding the instantiate and destroy performance penalty.
      	</p>

      	<h3>ShootingStarManager.cs</h3>
{% highlight cs %}
using UnityEngine;
using System.Collections;

public class ShootingStarManager : MonoBehaviour
{
    public Transform shootingStarPrefab;
    public float maxTimeBetween = 5.0f;
    public float distanceFromPlayer = 80.0f;
    public float travelDistance = 60.0f;
    public int poolSize = 3;

    private GameObject player;
    private float currentTimeBetween = 5.0f;
    private float timer;

    // use a pool to reuse shooting star objects
    private GameObject[] shootingStars;

    private void Start()
    {
        player = GameObject.FindGameObjectWithTag(Tags.player);
        if (!player)
        {
            Camera[] cams = GameObject.FindObjectsOfType<Camera>();
            if (cams.Length > 0)
            {
                player = cams[0].gameObject;
            }
        }
        shootingStars = new GameObject[poolSize];
        for (int i = 0; i < poolSize; ++i)
        {
            shootingStars[i] = ((Transform)Object.Instantiate(shootingStarPrefab,
                               Vector3.zero, Quaternion.identity)).gameObject;
            shootingStars[i].SetActive(false);
        }
        currentTimeBetween = Random.Range(0.0f, maxTimeBetween);
    }

    private void Update()
    {
        timer += Time.deltaTime;
        if (timer >= currentTimeBetween)
        {
            timer = 0.0f;
            ExecuteShootingstar(ChooseFreeShootingStar());

            // execute shooting start randomly but within maxTimeBetween
            currentTimeBetween = Random.Range(0.0f, maxTimeBetween);
        }
    }

    private GameObject ChooseFreeShootingStar()
    {
        foreach (GameObject star in shootingStars)
        {
            if (!star.activeSelf)
            {
                return star;
            }
        }
        return null;
    }

    private void ExecuteShootingstar(GameObject shootingStar)
    {
        if (shootingStar == null) return;

        // spawn a shooting start distanceFromPlayer away from the player in a random
        // direction, but only above the player
        Vector3 direction = new Vector3(Random.Range(-1.0f, 1.0f), Random.Range(0.0f, 0.9f),
                            Random.Range(-1.0f, 1.0f));
        shootingStar.transform.position = player.transform.position + direction.normalized *
                                          distanceFromPlayer;

        // directions are relative to the player
        Vector3 directionToStar = shootingStar.transform.position
                                  - player.transform.position;
        Vector3 directionRight = Vector3.Cross(directionToStar, Vector3.up).normalized;

        // rotate the right vector so it points in an arc downwards
        Quaternion rotation = Quaternion.AngleAxis(Random.Range(10.0f, 170.0f),
                              directionToStar);

        ShootingStar starComponent = shootingStar.GetComponent<ShootingStar>();

         // ensure the shooting star travels downwards relative to the player's horizon
        starComponent.endPos = shootingStar.transform.position
                               + (rotation * directionRight).normalized * travelDistance;
        shootingStar.SetActive(true);
        starComponent.StartEffect();
    }
}

{% endhighlight %}

        <h3>ShootingStar.cs</h3>
{% highlight cs %}
using UnityEngine;
using System.Collections;

public class ShootingStar : MonoBehaviour
{
    class AnimationParameters
    {
        public Vector3 scale;
        public float starTransparency;
        public float trailTransparency;
        public float flareBrightness;
    };

    public Vector3 startPos;
    public Vector3 endPos;
    public float minFlareBrightness = 0.0f;
    public float maxFlareBrightness = 1.0f;

    // the shooting star grows and shrinks during travel
    public float growTime = 1.0f;
    public float shrinkTime = 1.5f;
    public Vector3 minScale = new Vector3(0.5f, 0.5f, 0.5f);
    public Vector3 maxScale = new Vector3(2, 2, 2);

    private TrailRenderer tr;
    private bool travelling = false;
    private bool growing = true;
    private float timer = 0.0f;
    private LensFlare flare;
    private Material starMat;
    private Material trailMat;
    private float origTrailTime;
    private AnimationParameters maxValues;
    private AnimationParameters minValues;

    private void Start()
    {
        flare = GetComponent<LensFlare>();
        startPos = transform.position;
        starMat = GetComponent<Renderer>().material;
        trailMat = GetComponent<TrailRenderer>().material;
        starMat.SetFloat("_Transparency", 0.0f);
        trailMat.SetFloat("_Transparency", 0.0f);
        flare.brightness = minFlareBrightness;
        tr = gameObject.GetComponent<TrailRenderer>();
        origTrailTime = tr.time;
        maxValues = new AnimationParameters() { scale = maxScale, starTransparency = 1.0f,
                    trailTransparency = 1.0f, flareBrightness = maxFlareBrightness };
        minValues = new AnimationParameters() { scale = minScale, starTransparency = 0.0f, 
                    trailTransparency = 0.0f, flareBrightness = minFlareBrightness };
    }

    private void Update()
    {
        if (travelling)
        {
            timer += Time.deltaTime;

            // move shooting star
            transform.position = Vector3.Lerp(startPos, endPos,
                                              timer / (growTime + shrinkTime));

            // setting the time parameter to 0 for a frame effectively restarts the trail
            // rendered, preventing the problematic trail between the end of a shooting
            // star and the start of a reused one here we restore it to its oroginal value
            if (tr.time != origTrailTime && timer != 0)
            {
                tr.time = origTrailTime;
            }

            if (growing)
            {
                InterpolateEffects(timer / growTime, minValues, maxValues);
            }
            else
            {
                InterpolateEffects((timer - growTime) / shrinkTime, maxValues, minValues);
            }

            // stop grow, begin shrink
            if (timer >= growTime)
            {
                growing = false;
            }

            // end effect
            if (timer >= growTime + shrinkTime)
            {
                timer = 0.0f;
                travelling = false;
                growing = true;
                gameObject.SetActive(false);

                // reset the trail renderer
                tr.time = 0;
            }

        }
    }

    private void InterpolateEffects(float t, AnimationParameters fromParameters,
                                    AnimationParameters toParameters)
    {
        transform.localScale = Vector3.Lerp(fromParameters.scale, toParameters.scale, t);
        starMat.SetFloat("_Transparency", Mathf.Lerp(fromParameters.starTransparency,
                         toParameters.starTransparency, t));
        trailMat.SetFloat("_Transparency", Mathf.Lerp(fromParameters.trailTransparency,
                          toParameters.trailTransparency, t));
        flare.brightness = Mathf.Lerp(fromParameters.flareBrightness,
                                      toParameters.flareBrightness, t);
    }

    public void StartEffect()
    {
        startPos = transform.position;
        travelling = true;
    }
}

{% endhighlight %}

    </div>
</div>