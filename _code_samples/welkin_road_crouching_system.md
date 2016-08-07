---
layout: default
title: Crouching system (C#)
category: welkin_road
---
<div class="content-inner">
    <div class="container">
        <h1>Crouching system</h1>

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
            I wanted to make crouching midair more realistic by making it seem like the player pulled their legs up. This requires shrinking the player capsule, then moving its center to move it back into position so it is in line with the camera. Because of these changes, standing up becomes more complicated.
      	</p>

      	<h3>MovementController.cs</h3>
{% highlight cs %}
using UnityEngine;

// a section of the script
public class MovementController : MonoBehaviour
{
    public float crouchHeight = 1.0f;

    private CharacterController characterController;
    private CollisionDetector crouchCollisionTop;
    private CollisionDetector crouchCollisionBottom;
    private bool crouching = false;
    private float heightDifference;
    private bool crouchedWhileAirborne = false;
    private bool couldNotStandUp = false;
    private float crouchCameraChange = 0;

    private void Start()
    {
        characterController = GetComponent<CharacterController>();
        crouchCollisionTop = GameObject.Find("collision_detector_top")
                             .GetComponent<CollisionDetector>();
        crouchCollisionBottom = GameObject.Find("collision_detector_bottom")
                             .GetComponent<CollisionDetector>();
        heightDifference = (characterController.height - crouchHeight);
    }

    private void Crouch()
    {
        crouching = true;

        // shrink the player capsule when crouching
        characterController.height = crouchHeight;

        if (characterController.isGrounded)
        {
            // because the player capsule is shrunk, move its center down by half the height
            // difference so that it's touching the floor
            characterController.center = new Vector3(0, -heightDifference / 2, 0);
        }
        else
        {
            // crouch in air by pulling legs up: move capsule's center up by half of
            // the height difference
            crouchedWhileAirborne = true;
            characterController.center = new Vector3(0, heightDifference / 2, 0);

            // since the capsule is now positioned higher, move the top collision
            // trigger up as well
            crouchCollisionTop.gameObject.transform.localPosition += new Vector3(0, heightDifference, 0);
        }
    }

    private void StandUp()
    {
        // remember that it wasn't possible to stand up so that it can be done later
        couldNotStandUp = crouchCollisionTop.IsColliding;
        if (couldNotStandUp) return;

        // because the crouch midair was different than on the ground, the stand up needs
        // to be adjusted accordingly
        if (crouchedWhileAirborne)
        {
            // move the top collision trigger back
            crouchCollisionTop.gameObject.transform.localPosition += new Vector3(0, -heightDifference, 0);

            if (!characterController.isGrounded)
            {
                // the player crouched in air, then stand uped before fully landing
                if (crouchCollisionBottom.IsColliding)
                {
                    // calculate remaining distance to the floor
                    float distanceToFloor = -(crouchCollisionBottom.CollidingObject.ClosestPointOnBounds(transform.position) - transform.position).y;
                    float change = heightDifference + controllerSkinWidth - distanceToFloor;

                    // move the whole player controller up
                    transform.Translate(Vector3.up * change);

                    // move the camera back to the position before the stand up
                    // so it can animate smoothly
                    crouchCameraChange = -change;
                }
            }
            else
            {
                // move player controller up to standing position then extend
                // legs back to the ground

                // since the capsule was moved up when crouching midair and will be reset
                // when ncrouching, move the whole player controller up to compensate
                transform.Translate(Vector3.up * heightDifference);

                // move the camera back to the position before the stand up so it can
                // animate smoothly
                crouchCameraChange = -heightDifference;
            }
            crouchedWhileAirborne = false;
        }

        // reset the capsule's center and height
        crouching = false;
        characterController.center = Vector3.zero;
        characterController.height = controllerHeight;
    }
}
{% endhighlight %}

    </div>
</div>