# Laser Beam Focusing Calculator

A Jupyter notebook for planning laser processing with **f-θ scan lenses**. It covers Gaussian, flat-top and ring (annular) beams. For a laser and lens you supply, it calculates:

- the focused spot diameter and Rayleigh range (depth of focus),
- the beam diameter at any axial offset from focus or at any working distance,
- the inverse case: where to place the workpiece to get a target spot size, including stage-knob turns,
- a caustic (beam envelope) plot around focus,
- process metrics: peak irradiance, dwell time, line energy, keyhole number *Ke*, dimensionless energy density *E\**, and a lumped temperature-rise estimate,
- a reference table of defocus distances for 2–200 % diameter growth.

The derivations, assumptions and sources for the optics are in [`Laser_Equations.pdf`](Laser_Equations.pdf) (M. Hassan, 2026). Equation numbers below match that document.

> **M3PL lab edition.** This branch (`m3pl-lab`) is the version used in the [Multiscale Metal Manufacturing Processes Lab (M3PL)](https://m3pl.me.wisc.edu/) at the University of Wisconsin–Madison. In addition to the public version on `main`, Section 5 converts the required defocus into **Z-stage knob settings** for the lab's stage (0.127 mm per increment, 25 increments per rotation). If you use a different stage, change those two values or use the `main` branch.

---

## Contents

- [Installation](#installation)
- [Quick start](#quick-start)
- [Inputs](#inputs)
- [Notebook sections](#notebook-sections)
- [Equations](#equations)
- [Assumptions and limitations](#assumptions-and-limitations)
- [References](#references)
- [Author](#author)
- [About M3PL](#about-m3pl)
- [License](#license)

---

## Installation

Requires **Python 3.10 or newer** (developed and tested on Python 3.11–3.13).

```bash
git clone <this-repo-url>
cd laser

python -m venv .venv
# Windows (PowerShell):  .venv\Scripts\Activate.ps1
# macOS / Linux:         source .venv/bin/activate

pip install -r requirements.txt
```

Dependencies (`requirements.txt`): `numpy`, `matplotlib`, `notebook`, `ipykernel`.

## Quick start

```bash
jupyter notebook Laser_Beam_Calculator.ipynb
```

1. Run **Section 1** (imports) and the **helper-functions** cell once.
2. Enter your laser, lens and process parameters in **Section 2**, then run it.
3. Run whichever of **Sections 3–8** answers your question. After changing Section 2, re-run it before re-running any later section.

To run every cell without opening the notebook (useful as a smoke test or in CI):

```bash
jupyter nbconvert --to notebook --execute Laser_Beam_Calculator.ipynb --output executed.ipynb
```

## Inputs

All inputs are set in **Section 2** unless noted otherwise.

| Variable | Unit | Description |
|---|---|---|
| `wavelength_nm` | nm | Laser wavelength *λ* (e.g. 1070 for Yb fiber lasers). |
| `beam_diameter_at_lens_mm` | mm | Beam diameter *D* at the lens: 1/e² for Gaussian beams, physical (outer) diameter for flat-top or ring beams. |
| `M2` | – | Beam quality factor *M²* (1.0 = ideal TEM₀₀). Measure it per ISO 11146. |
| `focal_length_mm` | mm | Effective focal length *f*. **Used in all equations.** |
| `working_distance_mm` | mm | Mechanical working distance *WD* to the focal plane. **Used only to report stage positions.** |
| `beam_profile` | – | `'gaussian'`, `'flat_top'` or `'ring'`. |
| `ring_inner_ratio` | – | Ring beams only: inner/outer diameter ratio *ε* (0–0.9). |
| `known_focus_diameter_um` | µm | Optional. The measured or datasheet focal spot. When set, it overrides the computed spot; recommended for imaged flat-top or ring beams. Leave as `None` to compute the spot. |
| `power_W` | W | Average laser power *P*. |
| `frequency_Hz` | Hz | Pulse repetition rate (0 = CW). Reserved; not used in any calculation yet. |
| `scan_speed_mm_s` | mm/s | Scan speed *v* (0 = stationary). |
| `dz_mm` / `WD_mm` | mm | **Section 4:** set exactly one. |
| `target_diameter_um` | µm | **Section 5:** desired spot size (`None` or `'focus'` = focal spot). |
| `minimum_increment_of_knob_mm`, `increments_per_rotation` | mm, – | **Section 5:** Z-stage knob resolution (default 0.127 mm × 25 = 3.175 mm per rotation). Converts Δz into whole knob rotations + increments and shows the diameter each nearby setting actually gives. |
| `eta`, `rho`, `Cp`, `k`, `T_liquidus`, `T_initial`, `mass_kg`, `dt_s` | SI | **Section 7:** absorptivity and material properties. Defaults are for 316L stainless steel. |

**Sign convention:** Δz > 0 means the workpiece is moved **further from the lens**, i.e. below focus. So `WD + Δz` is below focus and `WD − Δz` is above focus.

## Notebook sections

| § | Section | Output |
|---|---|---|
| 1 | Setup | Imports. |
| 2 | Enter your laser | Input parameters. |
| – | Helper functions | Spot size, Rayleigh range, forward/inverse propagation, peak irradiance. |
| 3 | Focused spot and Rayleigh range | *d₀*, *w₀*, *z_R*, depth of focus 2*z_R*, peak irradiance, dwell time, line energy. |
| 4 | Beam size at a given position | *d(z)* for a given Δz **or** WD, growth vs. focus, peak irradiance. |
| 5 | Where to place the sample | \|Δz\| for a target diameter, both valid WD positions, knob rotations and increments. |
| 6 | Caustic plot | Beam radius vs. Δz, with a secondary WD axis. |
| 7 | Keyhole number, energy density, ΔT | *Ke*, *E\**, lumped ΔT and a regime indication. Uses `target_diameter_um` from **Section 5**, so run Section 5 first. |
| 8 | Reference table | \|Δz\|, diameter and WD for 2, 10, 15, 30, 50, 100 and 200 % growth. |

### Example (defaults)

The defaults are λ = 1070 nm, D = 5 mm, M² = 1.1, f = 163 mm, WD = 193 mm, P = 200 W, v = 100 mm/s. They give:

| Quantity | Value |
|---|---|
| Focused diameter *d₀* | 48.85 µm |
| Rayleigh range *z_R* | 1.593 mm (depth of focus 3.185 mm) |
| Peak irradiance at focus | 2.13 × 10⁷ W/cm² |
| \|Δz\| for a 200 µm spot | 6.322 mm → WD = 186.68 mm or 199.32 mm |

---

## Equations

### Optics (verified against `Laser_Equations.pdf`)

All of these use the effective focal length *f*, not the working distance.

**f-θ mapping (Eq. 1).** Lateral spot position *x* for a scan angle *θ* in radians:

```math
x = f\,\theta
```

**Gaussian irradiance (Eq. 2).** *w(z)* is the 1/e² radius:

```math
I(r,z) = I_0(z)\,\exp\!\left[-\frac{2r^2}{w^2(z)}\right]
```

**Focused waist (Eqs. 3–4).** Assumes a collimated input of 1/e² diameter *D = 2w_L* at the lens:

```math
w_0 = \frac{M^2\lambda f}{\pi w_L} = \frac{2M^2\lambda f}{\pi D},
\qquad
d_0 = 2w_0 = \frac{4M^2\lambda f}{\pi D}
```

**Rayleigh range (Eq. 5).** The distance from focus at which the beam area doubles (diameter × √2):

```math
z_R = \frac{\pi w_0^2}{M^2\lambda}
```

**Forward problem: diameter at an axial offset (Eq. 6):**

```math
d(z) = d_0\sqrt{1+\left(\frac{z-z_0}{z_R}\right)^2}
```

**Inverse problem: offset for a target diameter (Eqs. 7–8).** Here *ε = d_target/d₀ − 1* is the fractional diameter growth:

```math
\Delta z = \pm z_R\sqrt{\left(\frac{d_\text{target}}{d_0}\right)^2-1}
         = \pm z_R\sqrt{(1+\varepsilon)^2-1}
```

**Diffraction-limited (Airy) spot for a uniform pupil (Eq. 9).** This is the **first-zero** diameter, *not* a 1/e² diameter:

```math
r_\text{Airy} = 1.22\,\frac{\lambda f}{D}, \qquad d_\text{Airy} = 2.44\,\frac{\lambda f}{D}
```

**Second-moment (D4σ) propagation for non-Gaussian beams (Eqs. 10–11):**

```math
d_\sigma(z) = d_{\sigma,0}\sqrt{1+\left(\frac{z-z_0}{z_{R,\sigma}}\right)^2},
\qquad
z_{R,\sigma} = \frac{\pi w_{\sigma,0}^2}{M^2\lambda}
```

**Defocus multipliers (PDF Table 1).** For a Gaussian beam at constant power, peak irradiance retention is *1/(1+ε)²*.

| Growth | (1+ε)² − 1 | √((1+ε)² − 1) = \|Δz\|/z_R | Peak-irradiance retention |
|---|---|---|---|
| +2 % | 0.0404 | 0.2010 | 96.1 % |
| +10 % | 0.2100 | 0.4583 | 82.6 % |
| +15 % | 0.3225 | 0.5679 | 75.6 % |
| +30 % | 0.6900 | 0.8307 | 59.2 % |
| +50 % | 1.2500 | 1.1180 | 44.4 % |
| +100 % | 3.0000 | 1.7321 | 25.0 % |
| +200 % | 8.0000 | 2.8284 | 11.1 % |

### Additional equations used in the notebook (not in the PDF)

**Ring (annular) beam spot.** For a uniform annular pupil with obscuration ratio *ε = D_inner/D_outer*, the first dark ring sits at the first root *v₁* of the equation below. The notebook interpolates *k(ε)* over ε = 0–0.9; *k(0) = 1.22* recovers the Airy disk.

```math
J_1(v) - \varepsilon J_1(\varepsilon v) = 0,
\qquad
d_\text{ring} = 2k(\varepsilon)\,\frac{\lambda f}{D},\quad k = \frac{v_1}{\pi}
```

**Peak irradiance.** For a Gaussian beam this is the on-axis peak. For flat-top or ring beams it is the mean irradiance over the spot.

```math
I_\text{peak,Gauss} = \frac{2P}{\pi w^2} = \frac{8P}{\pi d^2},
\qquad
\bar I_\text{flat-top} = \frac{4P}{\pi d^2}
```

**Dwell time and line energy:**

```math
t_\text{dwell} = \frac{d}{v}, \qquad E_\ell = \frac{P}{v}
```

**Keyhole number.** From Gan et al. (2021), with absorptivity *η*, thermal diffusivity *α = k/(ρC_p)* and beam radius *r₀*:

```math
Ke = \frac{\eta P}{(T_l - T_0)\,\pi\rho C_p\sqrt{\alpha\,v\,r_0^3}}
```

**Dimensionless energy density:**

```math
E^* = \frac{\eta P}{\rho C_p (T_l - T_0)\,v\,d^2}
```

**Lumped temperature rise.** This is a first-order energy balance over a user-chosen heated mass *m* and time *Δt*. It ignores conduction, radiation, convection and latent heat.

```math
\Delta T = \frac{\eta P\,\Delta t}{m\,C_p}
```

---

## Assumptions and limitations

- **Paraxial, aberration-free optics** in a homogeneous medium. Real f-θ lenses have field-dependent aberrations, so spots off-axis are larger.
- **f ≠ WD.** Always enter the *effective focal length* for *f*. WD only places the workpiece. Confirm the true focal plane experimentally, for example with a ramp test or a caustic scan.
- **M² is a measured quantity.** Use an ISO 11146 caustic measurement. A datasheet value is a best case.
- **Flat-top and ring beams.**
  - Without `known_focus_diameter_um`, the notebook uses the Airy (or annular) **first-zero** diameter as *d₀*. It then propagates that value with the Gaussian law, Eq. 6, using the entered *M²*. That is only an approximation: Eqs. 10–11 strictly require a **second-moment (D4σ)** diameter, and a flat-top beam's *M²* is generally much greater than 1.
  - For imaged flat-top spots, enter the delivered spot via `known_focus_diameter_um`. A flat-top profile also does not stay flat-top away from the image plane.
  - The flat-top irradiance figure is the *mean* over the spot. For an Airy-limited spot, the true on-axis peak is ≈ 3.7× higher.
- **Section 7 is a screening tool, not a process model.**
  - The regime bands (*Ke* < 1, 1–10, > 10; *E\** < 1, 1–10, > 10) are heuristics. Calibrate them against your own experiments.
  - The ΔT value depends directly on the arbitrary `mass_kg` and `dt_s`; a physically consistent choice for Δt is the dwell time *d/v*.
  - The default 316L properties are room-temperature values, and the absorptivity *η* depends on wavelength, surface finish and process.
- **Note on the example in the PDF:** the PDF example (§8) uses WD = 192 mm, while the notebook default is 193 mm. Set `working_distance_mm` to the value for your lens.

## References

1. A. E. Siegman, *Lasers*, University Science Books, 1986.
2. B. E. A. Saleh and M. C. Teich, *Fundamentals of Photonics*, 3rd ed., Wiley, 2019.
3. H. Kogelnik and T. Li, "Laser Beams and Resonators," *Applied Optics* 5(10), 1550–1567, 1966.
4. J. W. Goodman, *Introduction to Fourier Optics*, 3rd ed., Roberts & Co., 2005.
5. W. J. Smith, *Modern Optical Engineering*, 4th ed., McGraw-Hill, 2007.
6. D. L. Shealy and J. A. Hoffnagle, "Laser beam shaping profiles and propagation," *Applied Optics* 45(21), 5118–5131, 2006.
7. Z. B. Harris et al., "Design and characterization of telecentric f-θ scanning lenses for broadband terahertz frequency systems," *AIP Advances* 10, 125313, 2020.
8. Z. Gan et al., "Universal scaling laws of keyhole stability and porosity in 3D printing of metals," *Nature Communications* 12, 2379, 2021.
9. ISO 11146-1:2021, *Lasers and laser-related equipment — Test methods for laser beam widths, divergence angles and beam propagation ratios.*

## Author

**Mahmudul Hassan**, PhD Candidate, Mechanical Engineering, University of Wisconsin–Madison.
Multiscale Metal Manufacturing Processes Lab (M3PL), advisor: Prof. Frank Pfefferkorn.

- **Research:** laser-based manufacturing and laser surface polishing, high-speed X-ray imaging of melt-pool and keyhole dynamics (with Argonne National Laboratory, BIAS and TU Wien), high-power laser platforms with programmable beam profiles, computer vision for process monitoring and quality control, and finite-element simulation of machining.
- **Education:**
  - Ph.D., Mechanical Engineering, University of Wisconsin–Madison (2022–present)
  - M.S., Mechanical Engineering, Miami University, Ohio (2020–2022)
  - B.S., Mechanical Engineering, Bangladesh University of Engineering and Technology (BUET) (2015–2019)
- **Selected publications:**
  - P. Faue, L. Rathmann, M. Möller, M. Hassan, et al., "High-speed X-ray study of process dynamics caused by surface features during continuous-wave laser polishing," *CIRP Annals*, 2023.
  - M. Hassan, J. Ma, M. P. Jahan, "Numerical modeling and simulation of machining of 3D printed CFRP composite," *Manufacturing Letters*, 2022.
  - M. M. Hassan, J. Ma, M. P. Jahan, "A Comparative Numerical Investigation on Machining of Laminated and 3D Printed CFRP Composites," *ASME IMECE*, 2022.
  - N. M. Cococcetta, M. P. Jahan, et al. (incl. M. M. Hassan), "Sustainable Post-Processing of 3D Printed Thermoplastic CFRP Composites Using Cryogenic Machining," *Journal of Manufacturing Processes*, 2021.

**Links:**
- [Personal website](https://www.mahmudulhassan.com)
- [Résumé](https://www.mahmudulhassan.com/full_resume.html)
- [M3PL profile](https://m3pl.me.wisc.edu/staff/hassan-mahmudul/)
- [UW–Madison Graduate School feature](https://grad.wisc.edu/2024/08/26/mahmudul-hassan/)
- [LinkedIn](https://www.linkedin.com/in/mmahmudulh)
- Email: [mhassan22@wisc.edu](mailto:mhassan22@wisc.edu)

## About M3PL

The [Multiscale Metal Manufacturing Processes Lab](https://m3pl.me.wisc.edu/) in the Department of Mechanical Engineering at the University of Wisconsin–Madison is directed by **Prof. Frank Pfefferkorn**. The lab studies discrete metal part manufacturing processes and heat transfer in manufacturing. Its research areas are:

- friction stir welding and processing;
- solid-state additive–subtractive manufacturing (e.g. friction surfacing);
- multi-material laser powder bed fusion;
- laser remelting for polishing, structuring and hardening of surfaces.

Mechanical Engineering Building, 1513 University Avenue, Madison, WI 53706, USA.

## License

Released under the [Apache License 2.0](LICENSE).
