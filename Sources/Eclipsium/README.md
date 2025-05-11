# Eclipsium
(*I have signed an NDA on this project what i am sharing is everything i am allowed to, there are more things i have done during this project*)

At Housefire Games, I partnered with game designers, artists, and senior engineers to architect, optimize, and elevate core systems, gameplay mechanics, and development tools. To deliver and launch their debut game ECLIPSIUM

### Spline-Sync Mechanics

Objective: Create a seamless mechanic that ties together two distinct “body forms” (e.g. miniture planet and planet) so that player inputs and environmental interactions remain fluid when transitioning.

**Requirements Gathering & Design**

- Collaborated with level designers to define how each body form should react to world geometry, physics, and input.

- Sketched out flow diagrams showing player state transitions, required data hand-offs (pose, velocity, animation state), and potential edge cases (mid-air switches, collision overlaps).

**Implementation Highlights**

- State-Driven progresser Bridge: Built a lightweight C# class that listens for form-switch events and copies over t progress parameters in real time.

- Dual-Rig Sync: Extended Unity’s Spline system to mirror root transforms between two rigs, ensuring the secondary rig (e.g. celestial planet) perfectly overlaps the primary during blending.

- Smooth Interpolation: Wrote a coroutine-based smoothing routine that interpolates position, rotation, and scale over a configurable duration. This prevented jitter when moving between forms on complex geometry, and a nice play-feel.

- Edge-Case Handling: Added checks to detect if the target form’s collider would intersect world geometry; if so, the switch is deferred until a safe frame to avoid “stuck” states.

Results

- Achieved creating a feeling of instantaneous planet-shift.

- Improving player feedback and immersion.

![](/Sources/Eclipsium/Images/OldPlaneteriumPuzzle.gif) 

--- 


### Shader “Consumption” Rework & Effects
Objective: Overhaul the original “darkness” damage-over-time effect and enhance it with tentacle visuals for a more menacing feel.

**Core Gameplay Tuning**

- Redesigned the damage tick timing and radius falloff so that players could see and react to the encroaching darkness.

- Added anticipatory VFX cues (consumption by tendrils) before the consumption of darkness, giving clear visual telegraphing.

**Tentacle Additions**

- Created a runtime system to spawn tentacle meshes around the player’s periphery when darkness activated.

- Each tentacle spawns at different rotation around the player, the tendrils (shader) grow exponintaily based on the players reaction.

- Blended tentacle shader value in/out based on proximity, so they appear to “reach” and then recede if the player escapes the radius.


![](/Sources/Eclipsium/Images/Tendrils.gif) 
![](/Sources/Eclipsium/Images/tendrilsControls.png) 

--- 

### Custom spline, Mesh deforming & Gravity shifting

##### **Custom Spline - Core Classes & Data Structures**


- SplinePoint: Holds position, tangent handles, and a “roll” angle for twist control.

- SplineSegment: Connects two points with a cubic Bézier interpolation, exposes GetPoint(t), GetTangent(t), and GetNormal(t) methods.

- Spline: Manages an ordered list of points, builds an array of segments on any edit, and maintains a cached lookup table of uniform-length samples for fast runtime queries.

**Editor Integration**

- Leveraged Unity’s OnSceneGUI to draw handles in the Scene view—points can be clicked, dragged, or deleted.

- Custom inspector buttons (“Add Curve”, “Undo Last”) call repository commands to modify the underlying point list and auto-commit changes to version control.

##### Mesh Deformation Along Splines
- With the spline core in place, I built a deformation system that “bends” arbitrary meshes to follow your curves.

- Vertex Remapping Algorithm

- UV-Driven Projection: Each deformable mesh is textured with a special UV channel where the U coordinate is re-interpreted as the distance along our spline (0–1) and the V coordinate as the lateral offset.

- Sample Spline: For every vertex, I sample position = spline.GetPoint(u) and tangent = spline.GetTangent(u).

- Construct Frenet Frame: Using tangent and the spline’s up-vector (or “roll” to handle twisting), I compute a normal and binormal to form a full local coordinate frame.

- Apply Offset: The vertex’s original lateral offset (V) gets applied along binormal, so the mesh wraps smoothly around the path.

**Custom spline**

![](/Sources/Eclipsium/Images/ConveyorSpline.png) 


![](/Sources/Eclipsium/Images/CustomSpline.png)

##### Gravity Reorientation via Surface Normals
To create moments where the player walks “up” walls or “down” inverted slopes, I wrote a math routine that dynamically shifts the gravity vector to align with the spline’s surface normals.

- Normal Extraction: For a given player-attach point on a deformable mesh (for instance, where the player’s feet hit), I sample the nearest spline normal using our GetNormal(u) call.

- If the player straddles multiple splines (e.g. at a junction), I blend normals based on proximity to each path.

- Gravity Vector Rotation Desired Down: Set desiredDown = –sampledNormal.
Smooth Transition: Rather than snapping, I slerp from the current gravity direction to desiredDown over 0.25 s, so the world “rotates” gently beneath the player.

- Character Controller Integration: I feed the updated gravity vector into our custom controller (controller.ApplyGravity(gravityVector);) and rotate the player’s root so “up” always opposes gravity.

- Edge Cases & Stability
When transitioning mid-air, I clamp the interpolation speed to avoid flipping the camera too fast.

- If the spline normal deviates more than 90° (e.g. from floor to ceiling), I require a brief ground contact before rotating gravity, preventing disorienting flips.

![](/Sources/Eclipsium/Images/Conveyors.gif) 
