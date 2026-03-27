# MacroUtils for Simcenter STAR-CCM+ — Setup & Usage Guide

A practical guide to installing and using [MacroUtils](https://github.com/frkasper/MacroUtils),
a high-level Java API library that simplifies writing automation macros for Simcenter STAR-CCM+.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Building the JAR](#building-the-jar)
4. [Running a Macro](#running-a-macro)
5. [Example — Flow in a Pipe](#example--flow-in-a-pipe)
6. [Project Structure](#project-structure)
7. [Tips & Troubleshooting](#tips--troubleshooting)

---

## Prerequisites

| Requirement | Details |
|---|---|
| Simcenter STAR-CCM+ | Tested with 2602 (Build 21.02.007) |
| JDK | Use the one bundled with your STAR-CCM+ install (JDK 21 for 2602) |
| Gradle | **Not required** — the repo includes a Gradle wrapper (`gradlew`) |
| Internet access | Required on first build only (Gradle downloads itself) |

### Set up JAVA_HOME

Point your environment to the JDK bundled inside STAR-CCM+:

```bash
export JAVA_HOME=~/apps/starccm/starccm_2602/jdk/linux-x86_64/jdk21.0.8
export PATH=$JAVA_HOME/bin:$PATH
```

To make this permanent, add those lines to your `~/.bashrc` or `~/.zshrc`, then run:

```bash
source ~/.bashrc   # or source ~/.zshrc
```

Verify:

```bash
java -version    # should report 21.x.x
javac -version   # should report 21.x.x
```

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/frkasper/MacroUtils.git
cd MacroUtils
```

### 2. Locate the STAR-CCM+ API JARs

The build needs access to STAR-CCM+'s internal Java libraries. Find them:

```bash
find ~/apps/starccm/starccm_2602 -name "*.jar" 2>/dev/null | grep -v jdk | head -20
```

They are typically located at:

```
~/apps/starccm/starccm_2602/STAR-CCM+21.02.007/star/lib/java/platform/modules/ext/
```

Verify that `star.common.Simulation` is present in those JARs:

```bash
for jar in ~/apps/starccm/starccm_2602/STAR-CCM+21.02.007/star/lib/java/platform/modules/ext/*.jar; do
    if jar tf "$jar" 2>/dev/null | grep -q "star/common/Simulation"; then
        echo "Found in: $jar"
    fi
done
```

### 3. Update `build.gradle`

Open `build.gradle` in the root of the MacroUtils repo and change the `libsFolder` line to point
to the directory found above:

```groovy
// Before (default — expects a folder one level above MacroUtils):
libsFolder = file("${rootDir}/../libs_STAR-CCM+")

// After (point directly to your STAR-CCM+ JARs):
libsFolder = file("/home/<your_user>/apps/starccm/starccm_2602/STAR-CCM+21.02.007/star/lib/java/platform/modules/ext")
```

---

## Building the JAR

From inside the MacroUtils directory:

```bash
./gradlew build > log.install 2>&1
cat log.install   # check for BUILD SUCCESSFUL
```

On success, the JAR is produced at:

```
macroutils/build/libs/macroutils_2602_build_MMDD.jar
```

where `MMDD` is today's date (e.g., `macroutils_2602_build_0327.jar`).

---

## Running a Macro

### Option A — Batch mode (terminal, no GUI)

```bash
starccm+ -batch MyMacro.java \
  -classpath /path/to/macroutils_2602_build_MMDD.jar \
  myCase.sim
```

Use `-new` instead of a `.sim` file if your macro builds everything from scratch:

```bash
starccm+ -batch MyMacro.java \
  -classpath /path/to/macroutils_2602_build_MMDD.jar \
  -new
```

### Option B — Interactive GUI

1. Open STAR-CCM+ and load your `.sim` file
2. Go to **Tools → Options → Environment** and add the MacroUtils JAR to the **User Macro Classpath**
3. Go to **Tools → Macros → Play Macro** and select your `.java` file

---

## Example — Flow in a Pipe

`Demo1_Flow_In_a_Pipe.java` is a complete end-to-end example that demonstrates:

- Creating a 3D cylinder geometry from scratch
- Setting up regions and renaming surfaces
- Configuring a polyhedral mesh with prism layers
- Defining laminar, steady, incompressible physics
- Setting velocity inlet and pressure outlet BCs
- Adding a scalar scene and asymptotic stopping criterion
- Running the solver and exporting results

### Geometry

```
                                   L = 500 mm
  +---------------------------------------------------------------+
  |                                                               |
r * O(0,0,0)                                    --> flow          |
  |                                                               |
  +---------------------------------------------------------------+
  r = 20 mm, oriented along X axis
```

### Pipeline Overview

```
initMacro()          → Initialise MacroUtils and set simulation title
prep1_createPart()   → Create cylinder CAD geometry
prep2_createRegion() → Rename surfaces (inlet/outlet/wall) and create region
prep3_BCsAndMesh()   → Set mesh params, physics continua, BCs, generate mesh
prep4_setPost()      → Add scalar scene, monitor, stopping criteria
mu.run()             → Run the solver
mu.saveSim()         → Save the .sim file
mu.io.write.all()    → Export scenes and reports
```

### Key MacroUtils calls used

```java
// Create geometry
mu.add.geometry.cylinder3DCAD(radius, length, origin, unit, axis);

// Rename surfaces by regex
mu.get.partSurfaces.byREGEX("x0", true).setPresentationName(ud.bcInlet);

// Set boundary conditions
mu.set.boundary.asVelocityInlet(boundary, velocity, TI, TLS, temp);
mu.set.boundary.asPressureOutlet(boundary, pressure, TI, TLS, temp);

// Generate mesh
mu.update.volumeMesh();

// Add post-processing
mu.add.scene.scalar(objects, fieldFunction, unit, true);
mu.add.report.massFlowAverage(boundary, name, ff, unit, true);

// Stopping criteria
mu.add.solver.stoppingCriteria(monitor, StopCriteria.ASYMPTOTIC, 0.001, 50);
```

### Run the demo

```bash
starccm+ -batch Demo1_Flow_In_a_Pipe.java \
  -classpath ~/codes/utilities/macroUtils/macroutils/build/libs/macroutils_2602_build_MMDD.jar \
  -new
```

---

## Project Structure

```
MacroUtils/
├── build.gradle              ← build config; set libsFolder here
├── gradlew                   ← Gradle wrapper (no Gradle install needed)
├── macroutils/
│   ├── src/macroutils/       ← MacroUtils library source
│   └── build/libs/           ← compiled JAR output goes here
├── demos/                    ← worked example macros
│   ├── Demo1_Flow_In_a_Pipe.java
│   ├── Demo2_...
│   └── ...
└── wiki/                     ← additional documentation
```

---

## Tips & Troubleshooting

**`package star.common does not exist` during build**
→ `libsFolder` in `build.gradle` is not pointing to the correct STAR-CCM+ JAR directory.
Run the `find` + `jar tf` commands above to locate the right path.

**`java.lang.ClassNotFoundException: macroutils.MacroUtils` at runtime**
→ The MacroUtils JAR is not on the classpath. Pass it with `-classpath` on the command line,
or add it under **Tools → Options → Environment → User Macro Classpath** in the GUI.

**Boundary not found error**
→ Boundary names in the macro must exactly match names in your `.sim` file.
Add a print loop before your BC calls to list all boundary names:
```java
mu.get.regions.all(true).forEach(r ->
    r.getBoundaryManager().getBoundaries().forEach(b ->
        mu.io.say.msg("Boundary: " + b.getPresentationName())));
```

**Wrong JDK version**
→ Always use the JDK bundled with your STAR-CCM+ installation. Run
`find ~/apps/starccm -name "java" -type f` to locate it, then export `JAVA_HOME` accordingly.

---

## Further Resources

- [MacroUtils GitHub](https://github.com/frkasper/MacroUtils)
- [MacroUtils Wiki & FAQ](https://github.com/frkasper/MacroUtils/wiki)
- [Siemens Community — Java Macros](https://community.sw.siemens.com/s/topic/0TO4O000000YThBWAW/simcenter-starccm-java-macros)
- [CFD Online — STAR-CCM+ Forum](https://www.cfd-online.com/Forums/star-ccm/)
- STAR-CCM+ built-in API Javadoc: `<install_dir>/STAR-CCM+21.02.007/star/lib/java/platform/docs/`
