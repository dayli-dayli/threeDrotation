# threeDrotation

Two single-file, browser-based 3D tools for building intuition about rotations. Each page is a standalone HTML file using [three.js](https://threejs.org/) r147 (loaded from a CDN). There is no build step.

| File | What it is |
| --- | --- |
| [goniometer.html](goniometer.html) | **Goniometer Normal Viewer.** Drive the motors of a sample goniometer and see where the sample's surface normal points. |
| [index.html](index.html) | **Rotation Aiming Bench.** Turn a body with Euler angles until its nose points at a target. Compares Euler interpolation, slerp and closed-loop tracking, and shows gimbal lock. |

## Running

Open either HTML file in a modern browser (Chrome, Edge, Firefox or Safari). The first load needs internet access for three.js and the Google Fonts. To serve the files locally instead:

```sh
python -m http.server 8000
# then open http://localhost:8000/goniometer.html
```

In the 3D view, drag to orbit, scroll to zoom, and right-drag to pan. Both pages follow the system light or dark theme.

## Conventions (both pages)

- The lab/world frame is right-handed with **Z up**.
- Positive angles follow the **right-hand rule**: point your thumb along the axis, and your fingers curl in the positive direction.
- Directions are reported as unit vectors, a polar angle θ measured from +Z, and an azimuth φ measured from +X toward +Y.

---

## Goniometer Normal Viewer (`goniometer.html`)

The page models a stack of three rotary motors, listed from the lab up:

1. **Base**: turns the whole assembly about lab Z (±180°).
2. **Y arm**: rides on the base and tilts about its own Y axis (±90°). Its axis is lab Y while the base is at 0.
3. **Spin**: rides on the Y arm and turns the sample about the **sample normal** (±180°).

The sample orientation is

```text
R = R_Z(Base) · R_Y(Y arm) · R_n(Spin) · T        (base motor first)
n = R · ẑ = R_Z(Base) · R_Y(Y arm) · n_s
```

Here `T` is the mount tilt that takes ẑ onto `n_s`, the sample normal in the Y-arm frame. Without a mounting offset, `T` is the identity and `n_s` is +Z. The spin axis is `n_s` itself, so **spin never moves n**. It turns the in-plane mark, and anything else fixed to the sample, about n.

### Features

- **Motors.** Each motor card has a slider and a number field. Under **Axis setup** you can rename the motor, change its travel limits, or flip its sign to match your controller's direction. You can also change the axis vector of the Base and the Y arm. The spin axis is always the sample normal, so it has no axis vector to edit. **Home all motors** drives every motor back to 0.
- **Normal readout.** Shows the unit vector n, polar θ, azimuth φ, the in-plane mark direction, and each motor axis as it currently sits in the lab. With the default axes it also shows the closed form. Without a mount offset, that is
  n = (cos Base · sin Y, sin Base · sin Y, cos Y), so θ = |Y arm| and φ = Base (or Base + 180° when the Y arm is negative).
- **Sample mount offset.** A miscut or wedge tilts the normal off the Y arm's Z direction by the given angle, toward the given azimuth. The spin motor follows the tilted normal.
- **Unit sphere wire grid.** The normal n is a unit vector, and its tip sits on the unit sphere. The sphere has grid lines every **10°** in latitude and longitude. Every 30° line is drawn slightly stronger, and the equator strongest, so angles can be read off at a glance. (The sphere is drawn at a fixed scene radius, and that radius represents length 1.)
- **Red arrows d and e.** There are three arrows in total. You control the blue normal n with the motors, and each red arrow with its own angle sliders.
  - **d** is a unit vector from the origin, so its tip sits on the unit sphere, just like n's.
  - **e** starts at the tip of d and is always **perpendicular to d**, so it lies tangent to the unit sphere at d's tip. Its length is adjustable from 0.1 to 0.95 and is always less than 1.
  - You set d with a polar angle θ_d and an azimuth φ_d, measured in a frame you choose:
    - **Sample frame** (default). Both arrows are attached to the sample. θ_d is measured from n, and φ_d is measured around n starting at the in-plane mark. Moving any motor, including spin, carries both arrows with the sample.
    - **Lab frame.** Both arrows are fixed in the lab: θ_d is measured from +Z and φ_d from +X. The motors move n but not d or e.
  - You set e with a single angle ψ_e, which turns e about d: e = cos ψ_e · ê_θ + sin ψ_e · ê_φ.
    - At ψ_e = 0, e points the way d's tip moves when θ_d increases.
    - At ψ_e = 90°, e points the way d's tip moves when φ_d increases.
  - Switching frames keeps both arrows where they are and recomputes their angles in the new frame.
  - **Align d with n** points d along the current normal.
  - The readout shows d and e in lab coordinates, the angles n ↔ d and d ↔ e, and the position of e's tip, d + L·e, along with its distance from the origin.
- **Per-motor paths.** Colored curves on the sphere show where n would travel if only the Base or only the Y arm turned. Each curve is solid within the motor's travel limits and dashed outside them. Spin has no curve, because it leaves n in place.
- **Trace.** Records the path of the normal's tip as you move the motors.
- **Snapshot.** **Save snapshot (PNG + TXT)** downloads two files that share a name: `<prefix>_YYYY-MM-DD_HH-MM-SS`.
  - **PNG:** the 3D view, with a caption stamped in the corner showing the normal's θ and φ, every motor angle, and the red-arrow settings.
  - **TXT:** the full configuration. That is every motor's angle, axis and limits; the mount offset; n with its θ and φ; the in-plane mark; d and e in lab coordinates; and the camera position.
  - The TXT ends with a single `JSON` line holding the same values (and the camera view) in machine-readable form. The spin motor is stored with `"axis": "normal"` and its sign.
  - The first time you save, the browser may ask permission to download more than one file.
- **Load snapshot.** **Load snapshot (TXT)**, or dropping a saved TXT onto the 3D view, reads that `JSON` line and restores the configuration. That covers every motor's angle and limits, the Base and Y-arm axes, the mount offset, both red arrows and their frame, and the camera. The rest of the file is ignored, so you can annotate it freely as long as the `JSON` line stays intact.
  - Older snapshots also load. Base and Y arm are matched by name, and an old **Stage** motor (which turned about the arm's Z) becomes Spin. Without a mount offset, Stage and Spin are the same rotation.
  - An old **X arm** is dropped. If it was at a nonzero angle, the page warns that the normal and the arrows may differ from the original.

### Derivation: rotating the red arrow d onto the +Z axis

**Problem.** The primary red arrow d has tilt θ_d and azimuth φ_d in the sample frame. Find the two rotations that turn d so it points along lab +Z: first a rotation about the **lab Y axis** (the Y arm), then a rotation about the **sample's own z axis**, its normal (Spin). The base is left at 0 and plays no part.

**Setup.** Take the default **sample frame** with no mount offset. At home (all motors at 0) the sample frame coincides with the lab frame: x is the in-plane mark, and z is the normal n. The arrow is then

$$
\mathbf{d}_s = \begin{pmatrix} \sin\theta_d \cos\varphi_d \\ \sin\theta_d \sin\varphi_d \\ \cos\theta_d \end{pmatrix}.
$$

The page uses right-handed rotations. About the lab axes,

$$
R_Y(\beta) = \begin{pmatrix} \cos\beta & 0 & \sin\beta \\ 0 & 1 & 0 \\ -\sin\beta & 0 & \cos\beta \end{pmatrix},\qquad
R_Z(\psi) = \begin{pmatrix} \cos\psi & -\sin\psi & 0 \\ \sin\psi & \cos\psi & 0 \\ 0 & 0 & 1 \end{pmatrix},
$$

and $R_{\mathbf{n}}(\psi)$ is a turn by ψ about a unit axis n.

#### Step 1: the geometric answer (shortest rotation)

The smallest rotation that takes d onto ẑ turns about the axis perpendicular to both, by the angle between them:

$$
\mathbf{k} = \frac{\mathbf{d}_s \times \hat{\mathbf{z}}}{\lVert \mathbf{d}_s \times \hat{\mathbf{z}} \rVert} = \begin{pmatrix} \sin\varphi_d \\ -\cos\varphi_d \\ 0 \end{pmatrix},
\qquad
\cos\omega = \mathbf{d}_s \cdot \hat{\mathbf{z}} = \cos\theta_d \;\Rightarrow\; \omega = \theta_d .
$$

The axis k is horizontal, at azimuth φ_d − 90°. The rotation matrix follows from Rodrigues' formula, with $K$ the cross-product matrix of k:

$$
R_{\min} = I + \sin\theta_d\, K + (1 - \cos\theta_d)\, K^2 .
$$

The Y arm only tilts about one fixed axis (lab Y), so on its own it cannot turn about k. Step 2 adds the spin about the sample normal to make up the difference.

#### Step 2: Y arm first, then spin about the sample z axis

**Rotation 1: Y arm by β about lab Y.** The whole sample tilts. Its normal and the arrow move to

$$
\mathbf{n} = R_Y(\beta)\,\hat{\mathbf{z}} = \begin{pmatrix} \sin\beta \\ 0 \\ \cos\beta \end{pmatrix},
\qquad
\mathbf{d}' = R_Y(\beta)\,\mathbf{d}_s .
$$

**Rotation 2: Spin by ψ about the sample z axis**, which is now the tilted normal n:

$$
\mathbf{d} = R_{\mathbf{n}}(\psi)\, R_Y(\beta)\, \mathbf{d}_s ,
$$

and the goal is $\mathbf{d} = \hat{\mathbf{z}}$.

**Condition on β.** A spin about n keeps every vector's angle to n unchanged. The arrow starts at angle θ_d from the normal and stays there, so spin sweeps d around a cone of half-angle θ_d about n. For d to reach ẑ, ẑ must lie on that cone:

$$
\hat{\mathbf{z}} \cdot \mathbf{n} = \cos\beta = \cos\theta_d
\quad\Longrightarrow\quad
\beta = \pm\theta_d .
$$

**Condition on ψ.** A turn about the tilted normal is a turn about z carried through the tilt: $R_{\mathbf{n}}(\psi) = R_Y(\beta)\, R_Z(\psi)\, R_Y(-\beta)$. Substituting,

$$
R_{\mathbf{n}}(\psi)\, R_Y(\beta)\, \mathbf{d}_s = R_Y(\beta)\, R_Z(\psi)\, \mathbf{d}_s = \hat{\mathbf{z}}
\quad\Longleftrightarrow\quad
R_Z(\psi)\, \mathbf{d}_s = R_Y(-\beta)\, \hat{\mathbf{z}} .
$$

Spinning about z keeps the tilt and adds ψ to the azimuth, and undoing the tilt from ẑ lands in the sample's xz plane:

$$
R_Z(\psi)\, \mathbf{d}_s = \begin{pmatrix} \sin\theta_d \cos(\varphi_d + \psi) \\ \sin\theta_d \sin(\varphi_d + \psi) \\ \cos\theta_d \end{pmatrix},
\qquad
R_Y(-\beta)\, \hat{\mathbf{z}} = \begin{pmatrix} -\sin\beta \\ 0 \\ \cos\beta \end{pmatrix}.
$$

Matching components: the y component gives φ_d + ψ = 0° or 180°, and the x component, $\sin\theta_d\cos(\varphi_d+\psi) = -\sin\beta$, picks which one goes with each sign of β. There are two solutions:

$$
\boxed{\ \beta = +\theta_d,\quad \psi = 180^\circ - \varphi_d\ }
\qquad\text{or}\qquad
\boxed{\ \beta = -\theta_d,\quad \psi = -\varphi_d\ }
$$

In words: tilt the Y arm by θ_d (either way), which puts +Z on the cone that d sweeps around the normal. Then spin the sample about its normal until d reaches +Z. Wrap ψ into ±180° if needed.

**Relation to Step 1.** For the second solution, the combined rotation is

$$
R_{\mathbf{n}}(-\varphi_d)\, R_Y(-\theta_d) = R_Y(-\theta_d)\, R_Z(-\varphi_d) = R_Z(-\varphi_d)\; R_{\min},
$$

so the two motors do the shortest rotation followed by a twist about +Z. That twist does not move d (it already points along +Z). It only fixes where the in-plane mark ends up.

**Limits.** For θ_d ≤ 90°, |β| = θ_d stays within the ±90° travel of the Y arm. Past 90° (d points below the sample plane), the arm would have to go past ±90°. With the default limits, d then cannot be brought onto +Z.

#### Worked example

θ_d = 35°, φ_d = 30°, so $\mathbf{d}_s = (0.4967,\ 0.2868,\ 0.8192)$.

| Solution | 1. Y arm β | 2. Spin ψ |
| --- | --- | --- |
| A | **35°** | 180° − 30° = **150°** |
| B | **−35°** | −30° = **−30°** |

To try it, leave the base at 0 and enter one of these in the page. **Arrow d (lab)** in the readout then shows (0, 0, 1), and its polar angle shows 0°. The page computes the pose from the motor values, so the order you type them in does not matter.

> **Mount offset.** With a mount offset (tilt t at direction a), the sample normal in the Y-arm frame is $\mathbf{n}_s = (\sin t\cos a,\ \sin t\sin a,\ \cos t)$, and the sample frame is pre-tilted by $T$, the shortest rotation taking ẑ onto $\mathbf{n}_s$. The same two steps apply.
>
> 1. The Y arm tilts the normal to $\mathbf{n} = R_Y(\beta)\,\mathbf{n}_s$. Spin then sweeps d around a cone of half-angle θ_d about n, so ẑ must lie on it: $\hat{\mathbf{z}}\cdot\mathbf{n} = \cos\theta_d$, i.e.
>
>    $$-\sin\beta\,\sin t\cos a + \cos\beta\,\cos t = \cos\theta_d .$$
>
>    With $A = \sqrt{\cos^2 t + \sin^2 t\,\cos^2 a}$ and $\delta = \operatorname{atan2}(\sin t\cos a,\ \cos t)$, this reads $A\cos(\beta + \delta) = \cos\theta_d$, so
>
>    $$\beta = -\delta \pm \arccos\!\left(\frac{\cos\theta_d}{A}\right).$$
>
>    A solution exists only when $|\cos\theta_d| \le A$.
>
> 2. Spin about n is a turn about ẑ in the sample frame, so with $\mathbf{u} = R_Y(-\beta)\,\hat{\mathbf{z}}$,
>
>    $$\psi = \operatorname{atan2}\!\big((T^{-1}\mathbf{u})_y,\ (T^{-1}\mathbf{u})_x\big) - \varphi_d .$$
>
> With no offset, A = 1 and δ = 0, and this reduces to β = ±θ_d. For example, t = 10°, a = 40° with the same arrow gives β = 26.79°, ψ = 159.66°, or β = −42.18°, ψ = −38.80°.
>
> **Lab frame.** In the lab frame the motors do not move d, so only the geometric rotation $R_{\min}$ of Step 1 applies.


---

## Rotation Aiming Bench (`index.html`)

A body with ROS REP-103 axes (x forward, y left, z up) sits in a three-ring gimbal. You turn it with intrinsic Euler angles until its nose (body x) points along a target direction t.

### Features

- **Target direction.** Set by azimuth and elevation, with presets for a random target, a target near the zenith, and a target directly behind the nose.
- **Euler angles.** Six intrinsic rotation sequences are available (Z-Y′-X″ yaw/pitch/roll, Y-X′-Z″, and others). The gimbal rings rebuild to match the selected sequence. **Play the sequence** animates the three rotations one at a time.
- **Solve for the target.**
  - *Aim level* builds a frame with roll = 0.
  - *Shortest arc* applies the smallest rotation that takes the nose onto t.
  - You can reach the solution by interpolating the Euler angles or by quaternion slerp. The nose path is drawn on the pointing sphere, and the page reports the swept angle against the great-circle minimum.
- **Closed-loop tracking.** Two control laws: a vector-error body-rate law ω = k·(f × t), and a per-angle gimbal servo. The target can stay still, circle in azimuth, or pass over the zenith, which shows why angle servos struggle near gimbal lock.
- **State.** Shows the pointing error, the target in body coordinates, the quaternion, the axis–angle form, the gimbal margin (90° − |angle 2|) and the full rotation matrix.

---

## Project layout

```text
goniometer.html   Goniometer Normal Viewer (self-contained: CSS, HTML, JS)
index.html        Rotation Aiming Bench (self-contained: CSS, HTML, JS)
README.md         This file
```

Each file follows the same structure: theme tokens in `:root`, then the control panel markup, then one script with the math, the scene construction, the UI wiring and the render loop.
