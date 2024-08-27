# Transforms

## Background

:::{figure-md} Slicer-coordinate-systems

![Coordinate systems](https://github.com/Slicer/Slicer/releases/download/docs-resources/coordinate_systems.png)

World coordinate system (left), Anatomical coordinate system (middle), Image coordinate system (right) Image source: [Slicer Coordinate Systems documentation](https://slicer.readthedocs.io/en/latest/user_guide/coordinate_systems.html)
:::

In 3D Slicer, medical image data is processed using the Right-Anterior-Superior (RAS) coordinate
system by default. However, to maintain compatibility with other medical imaging software, Slicer
assumes that files are stored in the Left-Posterior-Superior (LPS) coordinate system unless
otherwise specified. When reading or writing files, Slicer may need to flip the sign of the first
two coordinate axes to convert between RAS and LPS. The following transformation
matrix converts between RAS and LPS coordinates:

```{image} tmp_path/transforms_LPS_to_RAS_coords.svg
:align: center
:alt: LPS to RAS Coordinate Transformation
```

```{image} tmp_path/transforms_XYZ_axes.svg
:align: center
:alt: XYZ Axes
```

For more details on the coordinate systems used in 3D Slicer, refer to the [Slicer documentation](https://slicer.readthedocs.io/en/latest/user_guide/coordinate_systems.html).

### Spatial referencing

Spatial referencing data is used to encode voxel to world unit resolution (also referred to as pixel/voxel spacing), origin,
and orientation. In DICOM format, the spatial referencing can be retrieved from the DICOM header meta data. In MATLAB,
the [`imref3d`](https://www.mathworks.com/help/images/ref/imref3d.html) function can be constructed from the DICOM meta data to store the intrinsic spatial relationship.

Transforming between image space and world space results in proportional changes in the image volume:

```{image} tmp_path/transforms_Image_to_world_space.svg
:align: center
:alt: Image space to World Space
```

The matrix shown above can be used to convert from image space to world space, where `P_{x,y,z}` represents pixel spacing along
each axis, and `O_{x,y,z}` is the origin for each respective dimension.

When importing DICOM data into Slicer, a spatial referencing transform is automatically applied to a CT image volume based on the
header metadata in the DICOM header. Note the RAS anatomical orientation and +X, +Y, +Z (red/green/blue) axes indicators.

```{image} tmp_path/transforms_Slicer_DICOM_data.png
:align: center
:alt: DICOM Data Loaded into Slicer
```

Spatial referencing information of a volume in Slicer can be viewed in the Volume Information drop down of the
Volumes module.

```{image} tmp_path/transforms_Volume_spatial_info.png
:align: center
:alt: Spacial Referencing Information of a Volume
```

## Transforms in SlicerAutoscoperM

SlicerAutoscoperM enables tracking of rigid bodies in the 'World' space. When tracking multiple
rigid bodies, their relative motions can be computed with respect to a reference body.

In the AutoscoperM Slicer module, under the Pre-processing tab, rigid bodies of interest can
be automatically segmented from a CT DICOM volume. Alternatively, previously generated segmentations
can be loaded. These segmentations are used to generate partial volumes, where the density data
within the bounds of the segmented model is isolated and saved as a TIFF stack. In Autoscoper, this
TIFF stack is imported in a specific orientation, referred to as Autoscoper (AUT) space. The data from the
partial volume is projected onto the target image plane (overlaid with the radiograph for each frame)
as a Digitally Reconstructed Radiograph (DRR).

Like CT DICOM volumes, TIFF stacks are initially in image space and require a
spatial referencing transform to describe their world location, spacing, and
orientation. In Autoscoper, the voxel resolution of TIFF volumes follows a specified axes.

```{image} tmp_path/transforms_Model_and_its_TIFF_pv.svg
:align: center
:alt: Segmented Model and Corresponding TIFF Stack
```

To facilitate post-processing of output transforms from Autoscoper, several folders
are populated at the time of Partial Volume Generation in SlicerAutoscoperM. The
following structure outlines the information in the output directory:

```
Output Directory
│
├── Models
│   └── AUT{bone}.stl
│
├── Tracking
│   └── {bone}.tra
│
├── Transforms
│   ├── {bone}.tfm
│   ├── {bone}_DICOM2AUT.tfm
│   ├── {bone}_PVOL2AUT.tfm
│   ├── {bone}_scale.tfm
│   └── {bone}_t.tfm
│
└── Volumes
    └── {bone}.tif
```

* **Models:**
  * `AUT{bone}.stl`: Mesh file of the volume segmentation placed in Autoscoper (AUT) space.
* **Tracking:**
  * `{bone}.tra`: Equivalent to `{bone}_DICOM2AUT.tfm` but formatted for Autoscoper compatibility.
* **Transforms:**
  * `{bone}.tfm`: Non-rigid transform that translates and scales the `{bone}.tif` volume to its spatial location within the segmented CT-DICOM.
  * `{bone}_DICOM2AUT.tfm`: Transformation from DICOM space into Autoscoper (AUT) space.
  * `{bone}_PVOL2AUT.tfm`: Transformation from world space into Autoscoper (AUT) space.
  * `{bone}_scale.tfm`: Scaling matrix that converts the volume from image space to world space.
  * `{bone}_t.tfm`: Translation matrix moving between the world origin and the location of the partial volume within the segmented CT-DICOM.
* **Volumes:**
  * `{bone}.tif`: Non-spatially transformed volumetric data segmented from CT-DICOM.


:::{figure-md} Tfms-to-dicom-space

![Transforms of TIFF Stack to DICOM Space](tmp_path/transforms_Tfms_to_DICOM_Space.svg)

Transforms to DICOM space: `{bone}.tfm` (pink arrow), `{bone}_t.tfm` (blue arrow)
:::


:::{figure-md} Tfms-to-aut-space

![Transforms of PV to AUT Space](tmp_path/transforms_Tfms_to_AUT_Space.svg)

Transforms to Autoscoper space: `{bone}_DICOM2AUT.tfm` (orange arrow), `{bone}_t.tfm` (blue arrow) and `{bone}_PVOL2AUT.tfm` (gray arrow)
:::


```{mermaid}
flowchart TD
    subgraph image_space["Image Space"]
      tiff["TIFF"]
    end
    subgraph dicom_space["DICOM Space"]
      transformed_pvol["Spatially located PV"]
    end
    subgraph world_space["World Space"]
      world_origin["World origin"]
      partial_volume["Partial Volume (PVOL)"]
    end
    subgraph aut_space["Autoscoper Space (AUT)"]
      model["Model"]
    end
    world_space -- "{bone}_t.tfm" --> dicom_space
    image_space -- "{bone}.tfm" --> dicom_space
    image_space -- "{bone}_scale.tfm" --> world_space
    dicom_space -- "{bone}_DICOM2AUT.tfm" --> aut_space
    world_space -- "{bone}_PVOL2AUT.tfm" --> aut_space
```
