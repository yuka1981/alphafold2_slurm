# AlphaFold 2 Running Script

This repository is only for **QCT DevCloud** environment. The DevCloud use **SLRUM** and **enroot** to run performance testing across different GPU models. The following notes takes `aarch64` platform as exmaple to demo how to setup and run AlphaFold 2 in QCT DevCloud.

## Preparation

- Make sure you already login **NGC** with **NGC CLI tool**.

- Clone this repo in your <work_dir>, the following table shows the each purpose of scripts.

    | Script       | Description                                          |
    | ------------ | ---------------------------------------------------- |
    | run_af.sub   | SLURM script involving envrionment setting           | 
    | run_slurm.sh | Submit script                                        |
    | run_msa.sh   | Run protein structure prediction without MSA results |
    | run_nomsa.sh | Run protein structure prediction with MSA results    |
    | run.sh       | Main script involving environment setting            |

- Clone AlphaFold 2 repo in your <work_dir>

    ```bash
    git clone https://github.com/google-deepmind/alphafold.git
    ```

- Copy all scripts and `Dockerfile` to the Alphafold 2 repository directory

    ```bash
    cp Dockerfile run_af.sub run_slurm.sh run_msa.sh run_nomsa.sh run.sh alphafold2
    ```

- (Optional) Copy the `fastas` folder to Alphafold 2 repository directory. This folder contains different length of protein squence files for testing performance of GPU models.

    ```bash
    cp -r fastas alphafold2
    ```

- (Optional) Copy the `msas` folder to Alphafold 2 repository directory. Only for inference process, no MSA process running.

    ```bash
    cp -r msas alphafold2
    ```

## Build docker image

```bash
cd <your_work_dir>
docker build -t <your_docker_image_tag_name> .

# For example:
docker build -t af2-jax0429-cuda12-aarch64 .
```

## Build `sqsh` file for enroot

There two ways to create `sqsh` file.

- If you build up docker image via Dockerfile in this repo, import from local docker image as following command.

    ```bash
    enroot import --output af2-jax0429-cuda12-aarch64.sqsh "dockerd://<your_image_name>"

    # For example:
    enroot import --output af2-jax0429-cuda12-aarch64.sqsh "dockerd://af2-jax0429-cuda12-aarch64"
    ```

- Import from NGC, where `$oauthtoken` is your **NGC key** 

    ```bash
    # For example:
    enroot import --output your_.sqsh 'docker://$oauthtoken@nvcr.io#nvidia/tensorflow:24.12-tf2-py3'
    ```

## Run AlphaFold 2

- Config settings in `run_af.sub` file to meet your environment

    ```bash
    ## Basic settings
    export BACKBONE_DIR="<your_work_directory_path>"
    export DATADIR="<alphafold_2_dateset_root_path>"
    export NGPU="<#_of_GPUs>"

    _cont_name="AlphaFold2_${SLURM_JOB_ID}"
    _cont_mounts="${DATADIR}:/dataset,${BACKBONE_DIR}:/work" 
    _cont_image="${BACKBONE_DIR}/<your_sqsh_file_name>"

    ...

    ## Run AlphaFold 2
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
            ./run_msa.sh # change <run_msa.sh> or <run_nomsa.sh> to run AlphaFold in which setting your want
        echo "RUNANDTIME_STOP $(date +%s)"
    ) |& tee "${BACKBONE_DIR}/result-${SLURM_JOB_ID}-$(date +%s).log"

    echo "AlphaFold2 STOP $(date +%s)"
    ```

- Submit job to **SLURM**  
    You can execute the following command manually

    ```bash
    export DATASET_DIR=/dataset

    sbatch -N 1 --gpus-per-node=1 --ntasks-per-node=1 -J alphafold2 --partition=<your_slurm_partition> run_af.sub
    ```

    Or use `run_slurm.sh` script, you should edit the related setting first to fit your environment.

    ```bash
    ./run_slurm.sh
    ```

