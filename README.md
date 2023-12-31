# ThaumatoAnakalyptor
![0.5 meter long segmentation of scroll 3](pictures/thaumato_0-5m_scroll3.png)

*0.5 meter long segmentation of scroll 3.*

## Overview
**ThaumatoAnakalyptor** is an advanced automatic segmentation pipeline designed for high-precision extraction of segmentations from ct scans of ancient scrolls with minimal human intervention.

### Concept
The core principle of ThaumatoAnakalyptor involves extracting 3D points on papyrus surfaces and grouping them into sheets. These sheets are then used to calculate a mesh that can be used for texturing the sheet's surface.

### Process
- **3D Derivative Analysis:** The method employs 3D derivatives of volume brightness intensity to detect papyrus sheet surfaces in CT scans. An intuitive explanation is that the side view of a sheet exhibits a bell curve in voxel intensities, allowing for precise detection of the sheet's back and front.
- **PointCloud Generation:** Local sheet normals are calculated from 3D gradients analysis of the scroll scan. By thresholding on the first and second 1D derrivative in sheet normal direction, the process identifies surface voxels/points, resulting in a detailed surface PointCloud volume of the scroll.

    ![PointCloud Volume Visualization](pictures/pointcloud_slice_scroll3.png)

    *Slice view of the PointCloud volume of scroll 3.*

- **Segmentation and Mesh Formation:** The PointCloud volume is split into subvolumes, and a 3D instance segmentation algorithm clusters the surface points to instances of sheet patches. A Random Walk procedure is employed to stitch patches together into a sheet of points containing the 3D position and their respective winding number. Using Poisson surface reconstruction, the sheet points are then transformed into a mesh.

    ![Sheet Point Cloud during Segmentation Process](pictures/bending.png)

    *Stitched sheet PointCloud during the segmentation process. One half winding.*

    ![Mesh Formation](pictures/thaumato_mesh_sample.png)

    *Sample mesh.*

## Implementation

### Mesh Processing
- The mesh is unrolled to generate UV coordinates.
- Sub-meshes are created for easier texturing.
- VC’s render pipeline textures the sheet's surface.

## Running the Code
This example shows how to do segmentation on scroll 3 (PHerc0332).

### Download Data
- **Scroll Data:**
    Download ```PHerc0332.volpkg``` into the directory ```<scroll-path>``` and make sure to have the volume ID ```20231027191953``` in the ```<scroll-path>/PHerc0332.volpkg/volumes``` directory. Place the ```umbilici/scroll_<nr>/umbilicus.txt``` and ```umbilici/scroll_<nr>/umbilicus_old.txt``` files into all the ```<scroll-path>/PHerc0332.volpkg/volumes/<ID>``` directories.

    *Note*:
        To generate an ```umbilicus.txt``` for a scroll, make sure to transform the coordinates from scroll coordinates ```x, y, z``` - where ```x,y``` is the tif 2D coordinates and ```z``` the tif layer number - into umbilicus coordinates ```uc``` with this formula:
        ```
        uc = y + 500, z + 500, x + 500 
        ```
- **Checkpoint and Training Data:**
    Checkpoint and training data can be downloaded from the private Vesuvius Challenge SFTP server under ```GrandPrizeSubmission-31-12-2023/Codebase/automatic segmentation/ThaumatoAnakalyptor```.

### Execute the Pipeline
- **Setup:**
    Download the checkpoint ```last-epoch.ckpt``` for the instance segmentation model and place it into ```ThaumatoAnakalyptor/mask3d/saved/train/last-epoch.ckpt```. Build and start the Docker Container:

    ```bash
    docker build -t thaumato_image -f DockerfileThaumato .
    ```
    The following command might need adjustments based on how you would like to access your scroll data:
    ```bash
    docker run --gpus all -it --rm -v $(pwd)/ThaumatoAnakalyptor/:/workspace/ThaumatoAnakalyptor thaumato_image
    ```
- **Precomputation Steps:**
    The precomputation step is expected to take a few days.

    The Grid Cells used for segmentation have to be in 8um resolution. ```generate_half_sized_grid.py``` is used for 4um resolution scans to generate Grid Cells in 8um resolution.
    ```bash
    python3 generate_half_sized_grid.py --input_directory <scroll-path>/PHerc0332.volpkg/volumes/20231027191953 --output_directory <scroll-path>/PHerc0332.volpkg/volumes/2dtifs_8um
    ```
    ```bash
    python3 grid_to_pointcloud.py --base_path "" --volume_subpath "<scroll-path>/PHerc0332.volpkg/volumes/2dtifs_8um_grids" --disk_load_save "" "" --pointcloud_subpath "<scroll-path>/scroll3_surface_points/point_cloud"
    ```
    ```bash
    python3 pointcloud_to_instances.py --path "<scroll-path>/scroll3_surface_points" --dest "<scroll-path>/scroll3_surface_points" --umbilicus_path "<scroll-path>/PHerc0332.volpkg/volumes/umbilicus.txt" --main_drive "" --alternative_ply_drives "" --max_umbilicus_dist -1
    ```

- **Segmentation Steps:** Additional details for each segmentation step are provided in the [instructions](ThaumatoAnakalyptor/instructions.txt) document.
    First, pick a ```--starting_point```.
    The first time the script ```Random_Walks.py```  is run on a new scroll, flag ```--recompute``` should be set to 1. This will generate the overlapping graph of the scroll. For subsequent runs, flag ```--recompute``` should be set to 0 to speed up the process. Flag ```--continue_segmentation``` can be set to 1 if there already is a previous segmentation with the same starting point that you would like to continue.
    ```bash
    python3 Random_Walks.py --path "<scroll-path>/scroll3_surface_points/point_cloud_colorized_verso_subvolume_blocks" --starting_point 3113 5163 10920 --sheet_k_range -3 3 --sheet_z_range -10000 40000 --min_steps 16 --min_end_steps 4 --max_nr_walks 300000 --continue_segmentation 0 --recompute 1 --walk_aggregation_threshold 5
    ```

    For subsequent continuations of a segmentation, for example:
    ```bash
    python3 Random_Walks.py --path "<scroll-path>/scroll3_surface_points/point_cloud_colorized_verso_subvolume_blocks" --starting_point 3113 5163 10920 --sheet_k_range -3 3 --sheet_z_range -10000 40000 --min_steps 16 --min_end_steps 4 --max_nr_walks 300000 --continue_segmentation 1 --recompute 0 --walk_aggregation_threshold 5
    ```

- **Meshing Steps:** 
    When you are happy with the segmentation, the next step is to generate the mesh. This can be done with the following commands:
    ```bash
    python3 sheet_to_mesh.py --path_base <scroll-path>/scroll3_surface_points/3113_5163_10920/ --path_ta point_cloud_colorized_verso_subvolume_main_sheet_RW.ta --umbilicus_path "<scroll-path>/PHerc0332.volpkg/volumes/umbilicus.txt" ;
    ```
    ```bash
    python3 mesh_to_uv.py --path <scroll-path>/scroll3_surface_points/3113_5163_10920/point_cloud_colorized_verso_subvolume_blocks.obj --umbilicus_path "<scroll-path>/PHerc0332.volpkg/volumes/umbilicus.txt" ;
    ```
    The ```scale_factor``` depends on the resolution of the scroll scan. For 8um resolution, the scale factor is 1.0. For 4um resolution, the scale factor is 2.0.
    ```bash
    python3 finalize_mesh.py --input_mesh <scroll-path>/scroll3_surface_points/3113_5163_10920/point_cloud_colorized_verso_subvolume_blocks_uv.obj --cut_size 40000 --scale_factor 2.0 
    ```

- **Texturing Steps:** 
    VC rendering is used to generate the textured sheet. Place the folder ```working_<starting_point>``` into the ```PHerc0332.volpkg``` directory. Enter ```working_<starting_point>```. Then run the following commands:
    ```bash
    export MAX_TILE_SIZE=200000000000
    ```
    ```bash
    export OPENCV_IO_MAX_IMAGE_PIXELS=4294967295
    ```
    ```bash
    export CV_IO_MAX_IMAGE_PIXELS=4294967295
    ```
    ```bash
    vc_render --volpkg <scroll-path>/PHerc0332.volpkg/ --volume 20231027191953 --input-mesh point_cloud_colorized_verso_subvolume_blocks_uv.obj --output-file thaumato.obj --output-ppm thaumato.ppm --uv-plot thaumato_uvs.png --uv-reuse --cache-memory-limit 150G
    ```
    ```bash
    vc_layers_from_ppm -v ../ -p thaumato.ppm --output-dir layers/ -f tif -r 32 --cache-memory-limit 150G
    ```

- **Resource Requirements:** RTX4090 or equivalent CUDA-enabled GPU with at least 24GB VRAM, 196GB RAM + 250GB swap and a multithreaded CPU with >32 threads is required. NVME SSD is recommended for faster processing. Approximately twice the storage space of the initial scroll scan is required for the intermediate data.

### Training 3D instance segmentation (advanced)
If you would like to train the 3D instance segmentation model refer to [Mask3D](ThaumatoAnakalyptor/mask3d/README.md) and [these additional instructions](ThaumatoAnakalyptor/mask3d/install_commands.txt).
Make sure to download the provided training data ```3d_instance_segmentation_training_data```, preprocess the data and place it in the appropriate directories.

### Debugging
- **3D Visualization:** For debugging and visualization, CloudCompare or a similar 3D tool is recommended.

## Disclaimer
- Paths in the script might need adjustment as some are currently set for a specific system configuration.

## Known Bugs
- Recto and verso namings are switched.
- Starting points should be and the very bottom side of the sheet in a z slice trough the scroll.

## Contribution and Support
- As this software is in active development, users are encouraged to report any encountered issues. I'm happy to help and answer questions.

![Author](pictures/Author.jpg)

**Happy Segmenting!**
