---
layout: post
title: Moving an object over a multi line path using HoloLens 2 hand tracking
date: 2021-04-09T00:00:00.0000000+02:00
categories: []
tags:
- HoloLens2
- MRKT2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2021-04-09-Moving-an-object-over-a-multi-line-path-using-HoloLens-2-hand-tracking/movingoverline.png
comment_issue_id: 377
---
In [the previous part](https://localjoost.github.io/Basic-hand-gesture-recognition-with-MRTK-on-HoloLens-2/) we established how we could do some basic hand tracking and gesture recognition. The goal, at least part one of the goal, was to be able to move an object over a path existing of multiple way points - kind of a train on a track following your hand. Following your hand, but *not* getting off the track - or jumping from one part of the track to the other, skipping a way point. What we want, is this:

<iframe width="650" height="365" src="https://www.youtube.com/embed/YYInlW3kaDk" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Finding a point closest to a multi line

### Finding a point on a line segment

If you think I always write all my code myself, think again. I found [this code on gamedev.stackexhange](https://gamedev.stackexchange.com/questions/172001/how-to-detect-least-distance-to-line-segments-calculation). The first method is a near 100% copy:
```csharp
public static Vector3 GetClosestPointOnLineSegment(
    this Vector3 point, 
    Vector3 segmentStart, Vector3 segmentEnd)
{
    // Shift the problem to the origin to simplify the math.    
    var wander = point - segmentStart;
    var span = segmentEnd - segmentStart;

    // Compute how far along the line is the closest approach to our point.
    var t = Vector3.Dot(wander, span) / span.sqrMagnitude;

    // Restrict this point to within the line segment from start to end.
    t = Mathf.Clamp01(t);

    // Return this point.
    return segmentStart + t * span;
}
```
It accepts two points that make up a line segment (`segmentStart` and `segmentEnd`), and calculates the point that is closest to `point`

### Finding the closest line segment
Also mostly stolen code from the same post: how to find the closest segment of a multi line path. 
```csharp
public static int GetClosestLineSegmentIndex(this Vector3 point, List<Vector3> lineSegments)
{
    var closestIndex = -1;
    var closestSquaredRange = Mathf.Infinity;

    for (var i = 1; i < lineSegments.Count; i++)
    {
        var closestPoint = point.GetClosestPointOnLineSegment(lineSegments[i - 1],
                                                              lineSegments[i]);

        var squaredRange = (point - closestPoint).sqrMagnitude;

        if (squaredRange < closestSquaredRange)
        {
            closestSquaredRange = squaredRange;
            closestIndex = i - 1;
        }
    }
    return closestIndex;
}
```
I refer to the [original post](https://gamedev.stackexchange.com/questions/172001/how-to-detect-least-distance-to-line-segments-calculation) for and explanation of how this works.

Then I added this method to the series:
```csharp
public static Vector3 GetClosestPointOnLineSegment(this Vector3 point, List<Vector3> lineSegments, int index)
{
    return point.GetClosestPointOnLineSegment(
      lineSegments[index], lineSegments[index + 1]);
}
```
So when you want to now want to know which point is closest on a multi line, you first call `GetClosestLineSegmentIndex`, which gives you the segment, and then `GetClosestPointOnLineSegment` which gives you the actual point *on* that segment.

### Distance traveled along the path
Although we don't really need this now, I would like to know how far my point is from the start. This is the point of `DistanceTraveledAlongPath`. Given the fact we have calculated  `segmentIndex` using `GetClosestLineSegmentIndex` and `point` using `GetClosestPointOnLineSegment`, we can calculate the distance traveled along the path thus:
```csharp
public static float DistanceTraveledAlongPath(this Vector3 point,
                    List<Vector3> lineSegments, int segmentIndex)
{
    float distance = 0;
    if (lineSegments.Count > 1)
    {
        for (var i = 0; i < segmentIndex; i++)
        {
            distance += Vector3.Distance(lineSegments[i], lineSegments[i + 1]);
        }

        distance += Vector3.Distance(lineSegments[segmentIndex], point);
    }
    return distance;
}
```
And if we want to use the fraction of the path traveled, we can use the `TotalLength` method
```csharp
public static float TotalLength(this List<Vector3> lineSegments)
{
    float length = 0;
    if (lineSegments.Count > 1)
    {
        for (var i = 0; i < lineSegments.Count - 1; i++)
        {
            length += Vector3.Distance(lineSegments[i],
                                       lineSegments[i + 1]);
        }
    }

    return length;
}
```
And divide the result of `DistanceTraveledAlongPath` trough the result of this `TotalLength` method. This will give us any number between 0 (start) and 1 (end). 

## A base class for moving a point over a multi line path

### Settings 
In this demo we are using the point, that this class calculates using the earlier mentioned extension methods, to move an object over a set of way points. To that effect, we first define a base class that can be used for multiple purposes requiring a point following a hand over a line. It starts like this:

```csharp
public abstract class HandRailsBase : MonoBehaviour
{
    [Header("General Settings")]
    [SerializeField]
    private List<GameObject> wayPoints = new List<GameObject>();

    [SerializeField]
    private TrackedHandJoint joint = TrackedHandJoint.IndexMiddleJoint;

    [SerializeField]
    private bool trackRightHand = true;

    [SerializeField]
    private float maxDistanceFromLine = 0.1f;

    [SerializeField]
    private float maxDistanceFromWayPoint = 0.02f;
    
    [SerializeField]
    private bool trackPinch = true;

    [SerializeField]
    private bool trackGrab = true;    
```
The way points are define by a number of visible or invisible game objects - the choice is yours. I still track the IndexMiddleJoint like in the previous post, but this code suggests using one hand *or* the other, not both, otherwise things get pretty complicated. 

These two numbers are important:
* maxDistanceFromLine is the maximum distance your hand may deviate from the line before the point stops following you
* maxDistanceFromWayPoint is the maximum distance from a way point the projected point can jump from one segment to another. What I mean my this is best described using an image. 


![](/assets/2021-04-09-Moving-an-object-over-a-multi-line-path-using-HoloLens-2-hand-tracking/movingoverline.png)

We don't want to see the left situation. If the hand travels along the dotted line, we don't want to see the line following object suddenly jump from a to b. The user needs to follow the line, and only if the objects comes within maxDistanceFromWayPoint meters of the way point, it's allowed to cross over to the next (or previous) segment.

### Debug settings

```csharp
[Header("Debug Settings")]
[SerializeField]
private bool drawGizmoLines = false;

[SerializeField]
private Color gizmoLineColor = Color.yellow;

```
The behaviour sports some debug settings, which can be used to draw gizmos in the editor - this makes visualizing the path a bit easier while setting it up in the scene.

### Internal properties
```csharp
protected int CurrentIndex { get; private set; } = 0;

protected Vector3 PointOnLine { get; private set; } =
    Vector3.negativeInfinity;

protected List<Vector3> WayPointLocations { get; private set; }

protected float TotalLength { get; private set; }

private IMixedRealityHandJointService handJointService;
private IMixedRealityHandJointService HandJointService =>
    handJointService ??
        (handJointService =
          CoreServices.GetInputSystemDataProvider<IMixedRealityHandJointService>());
```
* `CurrentIndex` is the index of the currently active line segment, that is, the line segment the hand was closest to last time we looked
* `PointOnLine` is the currently projected point on the active line segment
* `WayPointLocations` is the list of points from `wayPoints`
* `TotalLength` is the total length of the path

These last two are calculated ahead so we don't have to do that time and time again:

```csharp
private void OnEnable()
{
    CurrentIndex = 0;
    WayPointLocations = wayPoints.Select(p => p.transform.position).ToList();
    TotalLength = WayPointLocations.TotalLength();
}
```
### Update - Moving along the multi line path
Basically the `Update` method does the rest, but I will go through it in parts. The first part is this:
```csharp
var trackedHand = trackRightHand ? Handedness.Right : Handedness.Left;
if (HandJointService.IsHandTracked(trackedHand) && 
    ((GestureUtils.IsPinching(trackedHand) && trackPinch) ||
     (GestureUtils.IsGrabbing(trackedHand) && trackGrab)))
{
    var jointPosition = 
       HandJointService.RequestJointTransform(joint,
                                              trackedHand).position;
    
    // Find closest point on the current segment
    PointOnLine = jointPosition.GetClosestPointOnLineSegment(
     WayPointLocations, CurrentIndex);
    if (Vector3.Distance(PointOnLine, jointPosition) >
        maxDistanceFromLine)
    {
        return;
    }
```
This piece of code:
* checks if the desired hand is tracked and pinching/grabbing as desired
* then finds the closest index and point on that index
* if that point is not further away from the hand than `maxDistanceFromLine` meters, we can go check if we can update the location.

The next part checks if we need to change segments, and starts with the check if we need to move a segment ahead:
```csharp
// Check if we should change to another line segment
var closestIndex = jointPosition.GetClosestLineSegmentIndex(WayPointLocations);
if (closestIndex != CurrentIndex)
{
    // Can we jump a segment forward?
    if (closestIndex == CurrentIndex + 1)
    {
        if (Vector3.Distance(WayPointLocations[CurrentIndex + 1], PointOnLine) <
            maxDistanceFromWayPoint)
        {
            CurrentIndex++;
            PointOnLine = jointPosition.GetClosestPointOnLineSegment(WayPointLocations,
               CurrentIndex);
        }
    }
```
It gets the index of the line *segment* closest to the hand position. If that is *not the current segment, we might need to jump a segment ahead or back. But we can only jump ahead (or back) *one*, for we *must* pass a way point before changing segment. 

Anyway, if the point on the *current* line segment is closer than `maxDistanceFromWayPoint` from the second way point of the active segment *and* the closest segment to the *hand* is indeed the *next* segment, we allow the point to 'cross over' to the next segment and calculate a point on *that*. 

If that is not the case, we might try to the same for moving *back*:

```csharp
else if (closestIndex == CurrentIndex - 1)
{
    if (Vector3.Distance(WayPointLocations[CurrentIndex], PointOnLine) <
         maxDistanceFromWayPoint)
    {
        CurrentIndex--;
        PointOnLine = jointPosition.GetClosestPointOnLineSegment(WayPointLocations, 
           CurrentIndex);
    }
}
```
And at the end of the `Update` method, we call the abstract method `OnLocationUpdated`, so child classes can do something with the updated location. 

### Little intermezzo: gizmos
In order to draw visualization thingies - in this case lines between the way points - you can implement a method like this.

```csharp
private void OnDrawGizmos()
{
    if (!drawGizmoLines)
    {
        return;
    }
    Gizmos.color = gizmoLineColor;
    var i = 0;
    while (i < wayPoints.Count - 1)
    {
        Gizmos.DrawLine(wayPoints[i].transform.position, 
          wayPoints[i + 1].transform.position);
        i++;
    }
}
```
The result is this:

![](/assets/2021-04-09-Moving-an-object-over-a-multi-line-path-using-HoloLens-2-hand-tracking/waypointgizmo.png)

Gizmos are graphic elements that are only displayed in the editor. They don't have any function - or appearance - in your final running app. 

## HandRails behaviour

Now the actual behaviour, a child class from HandRailsBase, that actually moves the green capsule over the line and show the progress information is just this:
```csharp
public class HandRails : HandRailsBase
{
    [SerializeField]
    private GameObject glider;

    [SerializeField]
    private TextMeshPro debugText;

    protected override void OnLocationUpdated()
    {
        glider.transform.position = PointOnLine;
        if (debugText != null)
        {
            var traveledLength =
                PointOnLine.DistanceTraveledAlongPath(WayPointLocations, 
                                                      CurrentIndex);
            var percentageTraveled = traveledLength / TotalLength * 100;
            debugText.text =
                $"TotalLength {WayPointLocations.TotalLength():F}\nDistance traveled {traveledLength:F}\n percentage {percentageTraveled:F}";
        }
    }
}
```
"`glider`" is the prefab that should be traveling alone the line, and `debugText` should be a `TextMeshPro` text where we can display some debug information. The rest is mere configuration

The HandRails behaviour sits in the HologramCollection, where it is configured like this:

![](/assets/2021-04-09-Moving-an-object-over-a-multi-line-path-using-HoloLens-2-hand-tracking/confighandrails.png)

The way points are simply yellow spheres, while the 'glider' is simply a green capsule with a TextMeshPro attached that travels along with it. That text is used to display some debug/progress information.

![](/assets/2021-04-09-Moving-an-object-over-a-multi-line-path-using-HoloLens-2-hand-tracking/glider.png)

## Some final words

Once you have the method to determine points on line and closest line segments, it's a matter of connecting the dots. The most tricky thing left is to determine when you are allowed to jump from line segment to line segment, but I think that's solved with this code pretty well.   
The [demo project for this blog can be downloaded here](https://github.com/LocalJoost/HandRailsDemo/tree/blog1).