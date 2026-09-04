# Building Energy

This workflow documents the full process used to convert HDB unit floorplans into EnergyPlus block models, validate the geometry and simulation setup, generate annual and short-duration scenarios, and compare different wall technologies.

The main model inputs are JSON floorplans and an EnergyPlus IDF template. The final outputs are EnergyPlus IDF and CSV files for different blocks, wall materials, and AC operating schedules.

---

## Stage 1: Generate unit-template mirrors
Create all required mirrored versions of the original HDB unit templates so they can be reused when assembling different block layouts.

### Input
- Original unit-template JSON file
- Each unit template contains:
  - room names
  - room coordinates
  - façade coordinates
  - window locations
  - corridor-facing walls
  - WWR information

### Process
For the standard HDB unit templates, generate three additional orientations:

- `mirror_x`
- `mirror_y`
- `mirror_o` / origin mirror

The transformations must be applied consistently to:

- all room coordinates
- window façades
- corridor façades
- any façade polyline points

After mirroring, coordinates should be cleaned so that:

- polygons do not contain duplicate points
- polygons do not retrace their own edges
- polygons remain valid
- room geometry is kept consistent between the original and mirrored versions

Save the resulting templates into a new JSON file, for example:
```
unittemplate_rm_mirror.json`
```
### Executive Maisonette exception
For Executive Maisonette units, mirrored templates do **not** need to be generated automatically.

The required Executive Maisonette unit arrangements are already defined explicitly and should be used directly.

### Output

```
unittemplate_rm_mirror.json
```

containing:

```
Original unit
Original + mirror_x
Original + mirror_y
Original + mirror_o
```
where required.

### Important checks

Before proceeding:
- plot every unit template
- check that room polygons are closed correctly
- check for room-area overlaps
- check for self-intersecting or retraced polygons
- check that windows and corridor façades remain on the correct exterior walls after mirroring

---

# Stage 2: Generate block floorplan
Combine the required unit templates into the actual floorplate of a HDB block.

### Input
- Unit-template JSON from Stage 1
- Required unit types for the block
- Unit numbers
- Orientation/mirror required for each unit
- Block footprint/layout information

Blocks developed include:
```
blk863b
blk864
blk863_EM
```

### Process

For each block:

1. Read the required unit templates from JSON.
2. Select the appropriate original or mirrored template.
3. Extract the required room geometry.
4. Rotate and translate units where required.
5. Assign a HDB unit number.
6. Combine all units into one block-level JSON floorplan.

Typical geometric operations include:

```
mirror
↓
rotate
↓
translate
↓
join units
↓
normalise coordinates
```
For Executive Maisonette blocks, the floorplate contains separate *upper* and *lower* layouts.

These alternate vertically when generating the full block:

```text
Storey 0 → lower
Storey 1 → upper
Storey 2 → lower
Storey 3 → upper
...
```

### Output

A block-specific JSON such as:

```text
blk863b.json
blk864.json
blk863_EM.json
```

### Important checks

Plot the complete floorplate and confirm:

- units are in their intended positions
- exterior façades are correct
- corridor-facing walls are correctly identified
- window façades are correctly oriented

This JSON becomes the main geometry input for all following stages.

Use
```
view_json.ipynb
```
to verify the HDB JSON output.

---

# Stage 3: Generate single-storey block for geometry and IDF testing
Generate a simplified EnergyPlus model before creating the full multistorey block.

This stage is mainly for debugging and geometry verification.

### Input
- Block floorplan JSON from Stage 2
- EnergyPlus template IDF
- EnergyPlus IDD

### Process
Convert every room polygon into an EnergyPlus thermal zone.

Each room is extruded vertically to create:
- floor
- ceiling
- walls

Typical floor-to-floor/zone height used: 2.8m

The building base elevation = 3.6m can also be included for HDB blocks with unoccupied void decks.

### Surface setup

#### Floor

```
Outside Boundary Condition = Adiabatic
Sun Exposure = NoSun
Wind Exposure = NoWind
```

#### Ceiling

```
Outside Boundary Condition = Adiabatic
Sun Exposure = NoSun
Wind Exposure = NoWind
```

#### Exterior wall

```
Sun Exposure = SunExposed
Wind Exposure = WindExposed
```

#### Corridor-facing wall

```
Sun Exposure = NoSun
Wind Exposure = WindExposed
```

### Windows

Windows are generated only on façades identified as:

```
type = window
```

The window area is determined from the specified WWR.

If a wall is both:

```
window + corridor
```

the wall remains window-bearing and corridor shading takes precedence:

```
Sun Exposure = NoSun
Wind Exposure = WindExposed
```

### HVAC setup

Cooling is provided only to:

- living rooms
- master bedrooms
- bedrooms

Other spaces remain without AC.

Cooling setpoint:

```
25 °C
```

No heating is required.

The system uses:

```
HVACTemplate:Zone:IdealLoadsAirSystem
```

### Infiltration

Typical infiltration assumptions:

#### Bedroom/master room

```
0.2 ACH
```

#### Living room with normal AC schedule

Use:

```
Living_Infiltration_AirCon
```

to represent:

```
AC OFF → approximately 1.0 ACH
AC ON  → approximately 0.2 ACH
```

#### Living room with 24hr AC

```
0.2 ACH
Always On
```

#### Other rooms

```
1.0 ACH
```

### Validation
Visualise the IDF and inspect each wall with
```
view_idf.ipynb
```

It is useful to colour walls according to their condition:
```
Window wall              → blue
Corridor wall            → green
Adiabatic/no-wind wall   → black
Other exterior wall      → red
```

### Output
Example:
```
blk863_EM_single.idf
```

This stage should be completed successfully before generating the full block.

---

# Stage 4: Generate annual multistorey HDB IDF templates
Expand the verified single-storey geometry into the complete multistorey block and create the two baseline annual simulation models.

### Input

- Validated block JSON
- EnergyPlus template IDF
- Number of storeys
- Storey height
- AC schedules

### Process

Repeat the floorplate vertically for the required number of storeys.

For normal HDB blocks, the same basic floorplate is copied and repeated.

For Executive Maisonette:

```
Even storey → lower layout
Odd storey  → upper layout
```

Each storey is vertically translated by:

```
2.8 m
```

### AC scenarios

Create two complete IDFs.

#### Normal AC

Uses:

```
AirCon_normal
```

for cooling availability.

Living-room infiltration also follows the AC-dependent infiltration schedule.

#### 24-hour AC

Uses:

```
AirCon_24hr
```

for cooling availability.

Living-room infiltration becomes:

```
0.2 ACH
Always On
```

### Annual outputs

Request cooling-energy outputs required for later analysis:

```text
Zone Ideal Loads Supply Air Total Cooling Energy
Zone Ideal Loads Supply Air Sensible Cooling Energy
Zone Ideal Loads Supply Air Latent Cooling Energy
```

Reporting frequency:

```text
Monthly
```

Monthly output makes annual post-processing much smaller and faster while still providing monthly trends.

### Output

Two baseline IDFs, for example:

```
blk863_EM_normal.idf
blk863_EM_24hAC.idf
```

### Validation
Run both baseline models first before generating all material variants.

Check:
- zero Severe Errors
- reasonable cooling loads
- correct number of zones
- correct number of storeys
- normal and 24 h AC schedules differ as expected
- all requested monthly outputs appear in the CSV

```
view_idf.ipynb
```
can also be used to verify the walls are assigned correct properties.

---

# Stage 5: Generate wall-material scenarios
Create separate annual EnergyPlus models to compare wall technologies.

### Input

Two baseline IDFs from Stage 4:

```
Normal AC
24 h AC
```

### Wall materials

Current scenarios:

```
ConcreteWall
WhitePaint
CoolPaint
EvaporativePaint
Greenwall
```

### Process

For each baseline IDF, change only the wall material of:

```
Project Wall
```

to the required material.

Do **not** change:

- roof
- floor
- ceiling
- windows
- geometry
- 
This allows the wall technology to be isolated as the primary comparison variable.

### Scenario matrix

```
                    ACnormal     AC24h
ConcreteWall           ✓           ✓
WhitePaint              ✓           ✓
CoolPaint               ✓           ✓
EvaporativePaint        ✓           ✓
Greenwall               ✓           ✓
```

Therefore:

```
2 AC schedules × 5 materials = 10 IDFs
```

### Folder structure

Example:

```text
blk864
│
├── annual
│   ├── blk864_ConcreteWall_ACnormal
│   ├── blk864_ConcreteWall_AC24h
│   ├── blk864_WhitePaint_ACnormal
│   ├── blk864_WhitePaint_AC24h
│   ├── blk864_CoolPaint_ACnormal
│   ├── blk864_CoolPaint_AC24h
│   ├── blk864_EvaporativePaint_ACnormal
│   ├── blk864_EvaporativePaint_AC24h
│   ├── blk864_Greenwall_ACnormal
│   └── blk864_Greenwall_AC24h
```

Each case folder should contain its corresponding:

```text
.idf
.csv
```

### Output

```text
10 annual IDF files
```

After running EnergyPlus:

```text
10 annual CSV files
```

### Annual post-processing

The main comparisons are:

- annual total cooling load
- annual sensible cooling load
- annual latent cooling load
- monthly cooling-load trends

Cooling energy is generally converted:

```text
J → MWh
```

using:

```text
1 MWh = 3.6 × 10^9 J
```

For comparison plots, ACnormal and AC24h should use the same Y-axis scale for the same cooling-load type.

---

# Stage 6: Generate three-day simulation scenarios

Create short-duration simulations for detailed hourly analysis of indoor temperature and cooling demand.

These simulations are primarily used to study short-term thermal behaviour that cannot be seen clearly from monthly annual outputs.

### Input

The same:

```text
2 AC schedules × 5 wall materials
```

from Stage 5.

### Weather file

Use the three-day weather file prepared for the study.

Example simulation period:

```text
8 Feb – 10 Feb 2018
```

The EnergyPlus `RunPeriod` should match the actual dates contained in the EPW.

### Process

For each of the 10 material/AC combinations:

1. copy the corresponding model
2. change the RunPeriod to the three-day period
3. use the three-day EPW
4. change the relevant outputs to:

```text
Hourly
```

### Temperature outputs

Request:

```text
Zone Mean Air Temperature
```

for selected living rooms.

Rather than outputting every temperature in the whole block, representative storeys can be selected.

For example:

```
Storey 0
Storey 4
Storey 8
```

for the Executive Maisonette model.

These levels allow comparison of:

- lower floor
- middle floor
- upper floor

### Cooling-load output

Request:

```text
Zone Ideal Loads Supply Air Total Cooling Energy
```

at hourly frequency.

### Folder naming

Example:

```text
blk864_ConcreteWall_ACnormal_3days
blk864_ConcreteWall_AC24h_3days
blk864_WhitePaint_ACnormal_3days
blk864_WhitePaint_AC24h_3days
...
```

For Executive Maisonette:

```text
blk863_EM_ConcreteWall_ACnormal_3days
blk863_EM_ConcreteWall_AC24h_3days
...
```

### Output

```text
2 AC schedules × 5 materials = 10 IDF files
```

After simulation:

```text
10 CSV files
```

### Three-day post-processing

The short simulations are used for several comparisons.

#### 1. Same room at different heights

Example:

```text
Unit 1 living room
Storey 0
Storey 4
Storey 8
```

Plot:

- hourly temperature trend
- temperature boxplot

Purpose:

Compare the effect of building height on indoor thermal conditions.

#### 2. Different units on the same storey

Example:

```text
All living rooms
Storey 4
ConcreteWall
ACnormal
```

Plot a boxplot to compare different unit positions/orientations within the same floorplate.

#### 3. Same room with different materials

Example:

```text
Unit 1
Storey 4
ACnormal
```

Compare:

```text
ConcreteWall
WhitePaint
CoolPaint
EvaporativePaint
Greenwall
```

using:

- hourly temperature trend
- temperature boxplot

Purpose:

Evaluate how wall technologies affect indoor thermal response.

#### 4. Total three-day cooling load

Compare:

```
5 materials × ACnormal
5 materials × AC24h
```

using total cooling energy over the three-day period.

---

# Overall workflow

```
Original Unit JSON
        │
        ▼
Stage 1
Generate unit mirrors
        │
        ▼
unittemplate_rm_mirror.json
        │
        ▼
Stage 2
Assemble units into block floorplan
        │
        ▼
Block JSON
        │
        ▼
Stage 3
Generate single-storey IDF
        │
        ├── Plot/check geometry
        ├── Check windows
        ├── Check corridor walls
        ├── Check surface matching
        └── Test EnergyPlus
        │
        ▼
Stage 4
Generate full multistorey block
        │
        ├── Normal AC IDF
        └── 24 h AC IDF
        │
        ▼
Stage 5
Apply 5 wall materials
        │
        ▼
10 Annual IDFs
        │
        ├── Run EnergyPlus
        └── Annual/monthly cooling-load analysis
        │
        ▼
Stage 6
Convert scenarios to 3-day simulations
        │
        ▼
10 Three-day IDFs
        │
        ├── Run EnergyPlus
        ├── Hourly indoor temperature
        └── Short-term cooling-load analysis
```

# Final project directory structure

A typical completed block should look approximately like:

```text
tampines_idf
│
├── blk863_EM
│   │
│   ├── blk863_EM_single.idf
│   ├── blk863_EM_normal.idf
│   ├── blk863_EM_24hAC.idf
│   │
│   ├── annual
│   │   ├── blk863_EM_ConcreteWall_ACnormal
│   │   ├── blk863_EM_ConcreteWall_AC24h
│   │   ├── blk863_EM_WhitePaint_ACnormal
│   │   ├── ...
│   │   └── blk863_EM_Greenwall_AC24h
│   │
│   ├── 3 days
│   │   ├── blk863_EM_ConcreteWall_ACnormal_3days
│   │   ├── blk863_EM_ConcreteWall_AC24h_3days
│   │   ├── ...
│   │   └── blk863_EM_Greenwall_AC24h_3days
│   │
│   └── results
│       ├── annual
│       └── 3 days
│
├── blk863b
└── blk864
```

# Key checks before passing a model to the next stage

The most important lesson is to **validate each stage before scaling it up**. A small geometry mistake becomes much harder to identify once it is repeated over many storeys and 10 simulation scenarios.

Before proceeding, check:
- Material inputs correct (project wall material correctly selected)
- Correct weatherfile (epw file) and RunPeriod
- Correct number of schedules:compact (AlwaysOn, AlwaysOff, Aircon_normal, Aircon_24hr, Living_Infiltration_AirCon)
- Correct schedules selected for 24hrs AC or normal AC hours
- No. of zones, windows, infiltration zones, and AC zones correct (100 rooms = 100 zones)
- Output:variable frequency correct (hourly or monthly)
- Infiltration zones ACH assigned 1 or 0.2 ACH correctly (especially living room)
- HVAC schedule correct (heating AlwaysOff, cooling depends on schedule, dehumidify at 60 and no humidifier, thermostat setpoint 25)

# Key Files

Before modifying the workflow, understand these main file groups. Remember to check and replace *file path* accordingly:
```
Stage 1
1. unittemplate_rm.json
3. unittemplate_EM.json
4. mirror_unittemplate.ipynb
5. view_json.ipynb

Stage 2
1. blk863_EM_script.ipynb, blk863b_script.ipynb and blk864_script.ipynb
2. idf_template_SL.idf
3. view_idf.ipynb

Stage 4
1. post_process_tampines.ipynb
```
