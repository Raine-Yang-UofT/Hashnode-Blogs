---
title: "Unity Dev Note: Simulated Bouncing for Kinetic GameObjects"
datePublished: Sat Jul 13 2024 19:43:47 GMT+0000 (Coordinated Universal Time)
cuid: clykjboxu000208ml9plc3zrb
slug: unity-dev-note-simulated-bouncing-for-kinetic-gameobjects
tags: unity, csharp

---

In this article, we will implements a simulated bouncing for 2D GameObjects with Kinematic `rigidbody2d`, based on method in [this video](https://www.youtube.com/embed/05eWA0TP3AA?enablejsapi=1&origin=https%3A%2F%2Fforum.unity.com)

### Enable Collision Detection

In Unity, `rigidbody` component has three types: dynamic, kinematic, and static. A dynamic rigidbody is influenced by all external forces, including gravity and forces exerted by other dynamic objects. A kinematic rigidbody can produce collision events (trigger callback functions `OnCollisionEnter`, `OnCollisionStay`, and `OnCollisionExit`) when colliding with other colliders, but are not influenced by forces. A static rigidbody behaves the same as a kinematic rigidbody whose position and rotation are frozen.

By default, a kinematic rigidbody only detects collisions with a dynamic rigidbody. To enable collision detections between kinematic and kinematic/static rigidbodies, we need to toggle on the option `Use Full Kinematic Contact`

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1720897625720/70bd562b-1488-4559-95ab-69c690843eb3.png align="center")

After that, a collision will be detected when two kinematic gameObjects come into contact. However, since the rigidbody types are still kinematic, the objects can move through each other without bouncing off. We need to create a custom script to simulate the bouncing.

### Custom bouncing script for kinematic objects

The key method we will use to implement the script is `rigidbody2D.Cast()`. This method is similar to ray collision detection, except that instead of detecting collision with a one-dimensional ray, it moves a copy of the gameObject's colliders to the given location and detects the collisions. In other words, it checks "if we move the gameObject to this location, what objects would it collide with". `Cast()` method has multiple overloads, and the one we use takes the following parameters:

```csharp
 public int Cast(Vector2 direction, ContactFilter2D contactFilter, RaycastHit2D[] results, float distance)
```

* direction: the direction to "move" the gameObjects' collider
    
* contactFilter: a filter that selects what collisions are included in the returned result. We can configure it to only check collisions with given layers.
    
* results: the collisions detected
    
* distance: the distance to "move" the gameObject's collider
    

With this method, the algorithm itself is fairly straightforward: when we try to move the gameObject in a given direction, we first use `Cast` to check whether there are obstacles blocking the gameObject. If not, we move the object.

```csharp
/*
* This method replaces the MovePosition method of the Rigidbody2D class
* to simulate collision detection and response for kinematic objects.
*/
public bool MovePosition(Vector2 direction, float moveSpeed) {
    int count = rb.Cast(
        direction.normalized,
        movementFilter,
        castCollisions,
        moveSpeed * Time.fixedDeltaTime + collisionOffset
    );

    if (count == 0) {
        // the movement is not blocked
        rb.MovePosition(rb.position + direction.normalized * moveSpeed * Time.fixedDeltaTime);
        return true;
    } else {
        return false;
    }
}
```

The method `MovePosition` above can be invoked as a replacement for `rigidbody2D.MovePosition` in the `FixedUpdate` method. Before it actually moves the object, it first performs a `Cast` that checks whether there are obstacles in the target position. If the direction is not blocked, it invokes `rigidbody2D.MovePosition` .

Below is the code for the complete script. Note that the key functionality lies solely in the `MovePosition` method, and the other codes are configurations specific for my own game project.

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MovementAgent : MonoBehaviour {

    Rigidbody2D rb;
    private List<RaycastHit2D> castCollisions = new List<RaycastHit2D>();
    public ContactFilter2D movementFilter = new ContactFilter2D();
    public float collisionOffset;     // an offset value to prevent the character from getting stuck in colliders


    // Start is called before the first frame update
    void Start() {
        rb = GetComponent<Rigidbody2D>();
        movementFilter.useLayerMask = true;
        movementFilter.layerMask = LayerMask.GetMask("ObstacleLayer");
    }

    /*
     * This method replaces the MovePosition method of the Rigidbody2D class
     * to simulate collision detection and response for kinematic objects.
     */
    public bool MovePosition(Vector2 direction, float moveSpeed) {
        int count = rb.Cast(
            direction.normalized,
            movementFilter,
            castCollisions,
            moveSpeed * Time.fixedDeltaTime + collisionOffset
        );

        if (count == 0) {
            // the movement is not blocked
            rb.MovePosition(rb.position + direction.normalized * moveSpeed * Time.fixedDeltaTime);
            return true;
        } else {
            return false;
        }
    }
}
```