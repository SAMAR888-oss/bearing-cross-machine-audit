# Datasets

None of the five benchmarks are included in this repository — they are large and
distributed by their owners. Download each one, then point the root constants in
the notebook's loader cells at your copies.

## CWRU

Case Western Reserve University Bearing Data Center. Drive-end accelerometer
`.mat` files, named by the Center's file numbers.

```
cwru/
  97.mat 98.mat 99.mat 100.mat     Normal baseline
  105-108, 169-172, 209-212        12k drive-end inner race
  130-133, 144-147, 156-161,       12k drive-end outer race (@3, @6, @12)
  197-200, 234-237
```

Each file holds a variable ending in `_DE_time`. Ball-fault files can be present;
the loader's file-number map skips them to keep the shared three-class space.

## Paderborn

Lessmeier et al., European Conference of the PHM Society, 2016.
Pinned to the single operating condition `N15_M07_F10`.

```
paderborn/
  K001/ K002/ ...     healthy    (K0*)
  KI01/ KI03/ ...     inner race (KI*)
  KA01/ KA03/ ...     outer race (KA*)
      N15_M07_F10_<code>_1.mat ... _20.mat
```

Channel index 6 inside the nested `Y` struct is `vibration_1`. `KB` codes
(combined damage) are skipped.

## MFPT

Machinery Failure Prevention Technology Society fault data sets.

```
mfpt/MFPT Fault Data Sets/
  1 - Three Baseline Conditions/baseline_{1,2,3}.mat
  3 - Seven More Outer Race Fault Conditions/OuterRaceFault_vload_*.mat
  4 - Seven Inner Race Fault Conditions/InnerRaceFault_vload_*.mat
```

Note the sampling-rate difference: baselines are 97656 Hz, faults 48828 Hz. The
reported results keep the native rates; `load_mfpt(resample_to=48828)` unifies
them if you want to check that difference is not driving separability.

## Ottawa (UORED)

Sehri, Dumond & Bouchard, *Data in Brief* 49:109327, 2023.

```
ottawa/3_MatLab_Raw_Data_Files (.mat)/
  1_Healthy/            H_*.mat
  2_Inner_Race_Faults/  I_*.mat
  3_Outer_Race_Faults/  O_*.mat
```

Each `.mat` holds a `(420000, 5)` array; column 0 is the drive-end vibration
channel used throughout. Ball and cage fault folders are unused.

## HIT-SM

Harbin Institute of Technology, Sensing and Measurement Laboratory.
<https://github.com/hitwzc/Bearing-datasets> — the SpectraQuest MFS rig only.

```
hitsm/SpectraQuest MFS dataset/
  Normal_900.mat  Normal_1200.mat  ...
  IR1_900.mat     IR2_900.mat      ...
  OR1_900.mat     OR5_1200.mat     ...
```

Class comes from the filename prefix; all shaft speeds are pooled by default.

---

Each dataset carries its own licence and terms of use. Check them before
redistributing anything derived from them.
