# AlphaFold 2 Execution Guide on QCT DevCloud

This repository provides scripts and instructions for running **AlphaFold 2** on the **QCT DevCloud** environment. QCT DevCloud utilizes **SLURM** as the job scheduler and **enroot** for containerized execution across different GPU models. This guide demonstrates how to set up and run AlphaFold 2 on the **AArch64** platform as an example. These instructions can be extended to other architectures by modifying the necessary environment settings and dependencies.

## Prerequisites

Before proceeding, ensure you have the following prerequisites met:

1. **Logged into NVIDIA GPU Cloud (NGC) using the NGC CLI tool.**
2. **Cloned this repository into your working directory (`<work_dir>`).**
3. **Cloned the AlphaFold 2 repository.**
4. **Installed Docker and Enroot for containerized execution.**
5. **Configured SLURM with appropriate permissions to submit jobs.**

### Repository Files

The following scripts are included in this repository and serve specific purposes:

| Script       | Description                                                   |
|-------------|---------------------------------------------------------------|
| `run_af.sub`   | SLURM submission script with environment setup              |
| `run_slurm.sh` | Script to submit SLURM jobs                                |
| `run_msa.sh`   | Runs protein structure prediction **without** MSA results  |
| `run_nomsa.sh` | Runs protein structure prediction **with** MSA results     |
| `run.sh`       | Main execution script including environment setup          |

### Clone AlphaFold 2 Repository

```bash
cd <work_dir>
git clone https://github.com/google-deepmind/alphafold.git
```

### Copy Required Files to AlphaFold Directory

To ensure proper execution, copy the necessary files into the AlphaFold 2 repository:

```bash
cp Dockerfile run_af.sub run_slurm.sh run_msa.sh run_nomsa.sh run.sh alphafold/
```

### (Optional) Copy Test Data

If you have test protein sequences or precomputed MSA results, copy them into the AlphaFold directory. You can also download protein FASTA files from the **Protein Data Bank (PDB)** website by following these steps:

1. Visit the PDB website: [https://www.rcsb.org/](https://www.rcsb.org/)
2. Use the search bar to find the protein of interest by entering the PDB ID or keyword.
3. Click on the protein entry to open its details page.
4. Navigate to the "Download" section and select the **FASTA Sequence** option.
5. Save the downloaded FASTA file and move it to your `fastas` directory for further processing.

```bash
cp -r fastas alphafold/  # Contains test protein sequences
cp -r msas alphafold/    # Contains precomputed MSA results (for inference only)
```

The `fastas` directory consists of protein sequences of varying lengths, enabling benchmark tests for different GPU models. The `msas` directory contains precomputed Multiple Sequence Alignment (MSA) results, which are useful when running inference without the need for MSA computation.

## Build the Docker Image

Navigate to your working directory and build the Docker image using the provided `Dockerfile`:

```bash
cd <work_dir>
docker build -t <your_docker_image_tag> .

# Example:
docker build -t af2-jax0429-cuda12-aarch64 .
```

The built image includes all necessary dependencies and libraries required to run AlphaFold 2 efficiently on QCT DevCloud. Be sure to name your image appropriately to avoid confusion when running multiple experiments.

## Create an Enroot SquashFS Image (`sqsh` File)

There are two ways to create an Enroot `sqsh` file:

### 1. Import from Local Docker Image

```bash
enroot import --output your_sqsh_file.sqsh "dockerd://your_docker_image_name"

# Example:
enroot import --output af2-jax0429-cuda12-aarch64.sqsh "dockerd://af2-jax0429-cuda12-aarch64"
```

### 2. Import from NVIDIA GPU Cloud (NGC)

This method is useful when utilizing prebuilt images from NGC, saving time on local builds. Ensure your NGC credentials are correctly configured before attempting the import.

#### Steps to Import an NGC Image

1. Execute the following **enroot import** command to fetch the desired container image from NGC:

    ```bash
    enroot import --output your_sqsh_file.sqsh 'docker://$oauthtoken@nvcr.io#nvidia/tensorflow:24.12-tf2-py3'
    ```

2. After running the command, the terminal will prompt for authentication:

    ```
    [INFO] Querying registry for permission grant
    [INFO] Authenticating with user: $oauthtoken
    Enter host password for user '$oauthtoken':
    ```

3. Enter your **NGC API Key** when prompted. This key is required to authenticate and access NGC-hosted container images.

4. Once the authentication is successful, enroot will proceed with downloading and converting the image into a `.sqsh` file, which can be used for execution.

## Running AlphaFold 2

### Configure `run_af.sub` for Your Environment

Edit the `run_af.sub` file to match your system settings. Below are descriptions of the necessary settings you need to configure:

`BACKBONE_DIR`: This is your main working directory where all execution files and logs are stored. Ensure this path is correctly set to a writable location.

`DATADIR`: This directory should contain the AlphaFold dataset files, including necessary model parameters and sequence databases. Make sure the dataset is correctly downloaded and structured as expected by AlphaFold.

`_cont_image`: This is the container image you built in the previous step. Ensure that the image path is correctly set to the `.sqsh` file generated via Enroot. If the image is not properly specified, the job may fail due to missing dependencies.

> [!NOTE]
> Choose the appropriate script of **line 55** of `run_af.sub` based on your use case:
>- `run_msa.sh`: Use this script if you want to perform MSA-based predictions. This mode extracts evolutionary relationships between sequences, improving accuracy at the cost of additional computation time.
>- `run_nomsa.sh`: Use this script if you prefer No-MSA-based predictions. This mode runs inference directly on the input sequence without generating MSA, resulting in faster execution but potentially lower accuracy.  
>
>Ensure that you select the correct script based on your dataset and computational requirements.

```bash
## Basic settings
export BACKBONE_DIR="your_work_directory_path"
export DATADIR="alphafold_2_dataset_root_path"
export NGPU=1

_cont_name="AlphaFold2_${SLURM_JOB_ID}"
_cont_mounts="${DATADIR}:/dataset,${BACKBONE_DIR}:/work"
_cont_image="${BACKBONE_DIR}/your_sqsh_file_name"

...

## Execute AlphaFold 2

echo "AlphaFold2 START $(date +%s)"
(
    echo "RUNANDTIME_START $(date +%s)"
    srun \
        --ntasks="$(( SLURM_JOB_NUM_NODES * NGPU ))" \
        --ntasks-per-node="${NGPU}" \
        --container-name="${_cont_name}" \
        --container-image="${_cont_image}" \
        --container-mounts="${_cont_mounts}" \
        --container-workdir=${WORK_DIR} \
        ./run_msa.sh  # Change to run_msa.sh or run_nomsa.sh as needed
    echo "RUNANDTIME_STOP $(date +%s)"
) |& tee "${BACKBONE_DIR}/result-${SLURM_JOB_ID}-$(date +%s).log"

echo "AlphaFold2 STOP $(date +%s)"
```

### Submit the Job to SLURM

#### Manually Submit Using `sbatch`

To manually submit the job using SLURM, execute:

```bash
export DATASET_DIR=/dataset
```

```bash
sbatch -N 1 --gpus-per-node=1 --ntasks-per-node=1 -J alphafold2 --partition=<your_slurm_partition> run_af.sub
```

This command allocates one node with a single GPU and submits the AlphaFold 2 execution job. Adjust parameters as needed based on your workload and available resources.

#### Using `run_slurm.sh`

Before execution, modify `run_slurm.sh` to match your environment. The script should be updated to reflect the manual SLURM submission command as follows:

```bash
export DATASET_DIR=/dataset

sbatch -N 1 --gpus-per-node=1 --ntasks-per-node=1 -J alphafold2 --partition=<your_slurm_partition> run_af.sub
```

Once the modifications are made, execute the script using:

```bash
./run_slurm.sh
```

The script automates job submission and ensures all necessary environment settings are correctly configured. Verify script settings before executing to prevent errors.

### Output Result

#### MSA-Based Prediction Result (Longer Wall Time)

![](/docs/images/figure1.png)

The figure above shows the inference process when using MSA-based predictions in AlphaFold 2. The total inference time, total MSA processing time, and the time taken for each structure prediction step are highlighted. MSA-based inference typically requires additional computation due to sequence alignment. The results include both per-structure inference times and the breakdown of MSA computation steps, which can be useful for analyzing computational efficiency across different hardware setups.

#### No-MSA-Based Prediction Example Result (Faster Execution)

![](/docs/images/figure2.png)

The provided figure showcases the inference time and breakdown of various steps when running AlphaFold 2 without MSA. The **total inference time** and time taken for individual structures are shown, allowing performance analysis across different configurations.

---

This guide provides a structured workflow for running **AlphaFold 2** on **QCT DevCloud** with **SLURM** and **Enroot**. By following these steps, you can efficiently set up, build, and run AlphaFold 2 for protein structure prediction. Modify the settings accordingly to match your infrastructure and ensure optimal execution for your specific use case. If you encounter any issues, refer to the AlphaFold 2 documentation or consult your system administrator for assistance.
