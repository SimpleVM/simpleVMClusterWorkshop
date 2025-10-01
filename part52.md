## Section 5 (Part 2): Scale up your analysis horizontally to further analyze the detected Microbiomes

In the previous part you found datasets that contain the **Staphylococcus Aureus** and the **Acinetobacter Baumannii** strain. 
In the second part of section 5 you will investigate your cluster setup and use the infrastructure
for your computations. We will then use the cluster to assemble the metagenomes and try to bin the strains in order to analyze the genes.

### 5.1 Investigate your cluster setup

> ⚠️ **Important:** Your **Cluster must have the status RUNNING** to perform the following steps.


1. Click on the Clusters tab. After you have initiated the start-up of the cluster,
   you should have been automatically redirected there. Now click on the cluster to open the dropdown.
   Click on the Theia IDE URL which opens a new browser tab.

2. Click on `Terminal` in the upper menu and select `New Terminal`.
   ![](figures/terminal.png)

3. Check how many nodes  are part of your cluster by using `sinfo`. 

```
sinfo  --Node -o "%N %m %c %S %t"
```
which will produce the following example output
```
NODELIST MEMORY CPUS ALLOCNODES STATE
bibigrid-master-7bg2m4zgp13jxhp 31962 14 all drain
bibigrid-worker-7bg2m4zgp13jxhp-0 64075 28 all idle
bibigrid-worker-7bg2m4zgp13jxhp-1 64075 28 all idle
bibigrid-worker-7bg2m4zgp13jxhp-2 64075 28 all idle
```
You can also see the number of cpus (CPUs column) and the amount of RAM (MEMORY column) assigned.
Another important columns here are `STATE` which tells you if the worker nodes are processing jobs
or are just in `idle` state and the column `NODELIST` which is just the name of the worker node.

4. You could now submit a job and test if your cluster is working as expected.
   **/vol/spool** is the folder which is shared between all nodes. You should always submit jobs
   from that directory.
   ```
   cd /vol/spool
   ```

5. Please fetch the script that we want to execute
   ```
   wget https://openstack.cebitec.uni-bielefeld.de:8080/simplevm-workshop/basic.sh
   ```
   The script contains the following content:
   ```
   #!/bin/bash
   
   #Do not do anything for 30 seconds 
   sleep 30
   #Print out the name of the machine where the job was executed
   hostname
   ```
   where
    * `sleep 30` will delay the process for 30 seconds.
    * `hostname` reports the name of the worker node.

6. You could now submit the job to the SLURM scheduler by using `sbatch` and directly after that
   check if SLURM is executing your script with `squeue`.

   sbatch:
   ```
   sbatch basic.sh
   ```
   
   squeue:
   ```
   squeue
   ```
   which will produce the following example output:
   ```
   JOBID PARTITION     NAME       USER    ST      TIME  NODES NODELIST(REASON)
   1     openstack     basic.sh   ubuntu  R       0:04      1 bibigrid-worker-7bg2m4zgp13jxhp-0
   ```
   Squeue tells you the state of your jobs and which nodes are actually executing them.
   In this example you should see that `bibigrid-worker-7bg2m4zgp13jxhp-0` is running (`ST` column) your job
   with the name `basic.sh`.

7. Once the job has finished you should see a slurm output file in your directory (Example: `slurm-212.out`)
   which will contain the name of the worker node which executed your script.
   Open the file with the following command:
   ```
   cat slurm-*.out
   ```
   Example output:
   ```
   bibigrid-worker-7bg2m4zgp13jxhp-0
   ```

8. One way to distribute jobs is to use so-called array jobs. With array jobs you specify how many times
   your script should be executed. Every time the script is executed, a number between 1 and the number of times
   you want the script to be executed is assigned to the script execution. The specific number is saved in a
   variable (`SLURM_ARRAY_TASK_ID`). If you specify `--array=1-50` then your script is 50 times executed and
   the `SLURM_ARRAY_TASK_ID` variable will get a value between 1 and 50. SLURM will distribute the
   jobs on your cluster.

   Please fetch the modified script
   ```
   wget https://openstack.cebitec.uni-bielefeld.de:8080/simplevm-workshop/basic_array.sh
   ```

   Which is simply reading out the `SLURM_ARRAY_TASK_ID` variable and placing them in a file in an
   output directory:

   ```
   #!/bin/bash
   
   # Create output directory in case it was not created so far
   mkdir -p output_array
   
   #Do not do anything for 10 seconds 
   sleep 10
   
   #Create a file with the name of SLURM_ARRAY_TASK_ID content. 
   touch output_array/${SLURM_ARRAY_TASK_ID}
   ```
 
   You can execute this script a 50 times with the following command 
   ```
   sbatch --array=1-50 basic_array.sh
   ```
   You could now check the progress using **squeue** and **watch**.
   ```
   watch 'squeue --format="%.18i %.90j %t %.6C %m %.10M %N"'
   ```
   Watch automatically gives you the output of a command at regular intervals. The default is 2 seconds.
   You can leave the output of **watch** by pressing `CTRL + C`

   ![](./figures/output_basic_array_watch.png)

   
   If you check the `output_array` folder when the jobs are finished, you should see numbers from 0 to 50.
   ```
   ls output_array
   ```
10. Remove the `slurm-{X}_{Y}.out`-files after the output is generated by running the following command:
   ```
   rm slurm-*
   ```
   This will clean up your `/vol/spool/`-directory for the next steps. 

### 5.2 Prepare the Metagenomics-Toolkit run  

The [Metagenomics-Toolkit](https://github.com/metagenomics/metagenomics-tk) will only run the steps quality control, assembly, binnning and classification on the three
detected datasets. Internally is the Metagenomics-Toolkit workflow a Nextflow based workflow and Nextflow is also using commands like sbatch behind the scenes. 
Especially for the classification part we need a lot of storage in order to store the GTDB database.

1. Create a database directory

   ```
   mkdir /vol/spool/database
   ```

2. We need to download the GTDB database from our object storage storage. This time we will use another S3 tool called s5cmd which is pre-installed on the VM. The following command will take up to 10 minutes.
   ```
   s5cmd  --endpoint-url https://s3-int.bi.denbi.de  --no-sign-request cp --concurrency 28  s3://databases/gtdbtk_r226_v2_data/release* /vol/spool/database
   ```
   While the command is running you could add one additional worker node via the SimpleVM website.
   On the website open the cluster tab and click on the "Scale Cluster up" button.
   ![](./figures/cluster_scale_up.png)

   On the opened modal specify that you want to add one worker node of the **de.NBI large + ephemeral** flavor.
   ![](./figures/scale_up_modal.png)

   The following modal will open once you confirm that you want to scale up the cluster:
   ![](./figures/scaling_commands.png)

   The first command will show you an SSH command that we will not use during the workshop, since we are using a research environment.
   The second command should only be executed after all workers are active (see screenshot below). Three out of three workers must be active.
   Please store the second command somewhere temporarily on your laptop. You will have to execute it once all workers are active.  
   ![](./figures/active_worker.png)
   
   All worker nodes must be active. Then you can open a new terminal in the research environment and run the following commands.

   ```
   cd ~/
   ```
   
   ```
   Now, execute the command that you have stored on your laptop. 
   ``` 
 
   Run `sinfo` to verify that you have now three nodes in your cluster.
   You can proceed as soon as the download in your other terminal window has finished.

3. Install Java 

   ```
   sudo apt install -y unzip default-jre 
   ```

4. Install Nextflow

   ```
   curl -s https://get.nextflow.io | bash
   ```

5. Move it to a folder where other binaries usually are stored:
   ```
   sudo mv nextflow /usr/local/bin/
   ```

6. Change file permissions:
   ```
   chmod a+x /usr/local/bin/nextflow
   ```

7. We need to tell the Toolkit how to access the data stored in S3

```
cat > /vol/spool/aws.config <<EOF
aws {
  client {
      s3PathStyleAccess = true
      connectionTimeout = 120000
      maxParallelTransfers = 28
      maxErrorRetry = 10
      protocol = 'HTTPS'
      connectionTimeout = '2000'
      endpoint = 'https://s3-int.bi.denbi.de'
      signerOverride = 'AWSS3V4SignerType'
    }
   }
EOF
```

8. We will fetch the Toolkit directly from GitHub. Due to pull rate restrictions we will have to provide a read-only access token which was generated only for this workshop. You can read more about this in our [wiki](https://simplevm.denbi.de/wiki/FAQ/#i-have-problems-downloading-packages-from-github-eg-in-r).

```
mkdir -p /vol/spool/.nextflow
```

```
cp ~/.nextflow/scm /vol/spool/.nextflow
```

### 5.3 Run the Toolkit

1. Go to the shared directory:
   ```
   cd /vol/spool
   ```

2. Create a file listing the SRA run ids you want to process: 
   ```
   echo -e "ACCESSION\nERR2683178\nSRR492065\nSRR6439514" > sra.tsv 
   ```
   
3. Run the Metagenomics-Toolkit. One of the datasets should be processed within a few minutes, so you can continue to the next subsection to inspect
   the result of the sample.
   ```
   NXF_HOME=$PWD/.nextflow NXF_VER=25.04.2 nextflow run metagenomics/metagenomics-tk \
        -r 0.13.2 \
        -c /vol/spool/aws.config \
        -ansi-log false \
        -profile slurm -resume -entry wFullPipeline \
        -work-dir work \
        -params-file https://raw.githubusercontent.com/SimpleVM/simpleVMWorkshopGCB/refs/heads/main/config/fullPipeline_illumina_nanpore.yml \
        --databases=/vol/scratch/database/ \
        --input.SRA.S3.path=/vol/spool/sra.tsv \
        --output=output \
        --steps.magAttributes.gtdb.database.extractedDBPath=/vol/spool/database/release226
   ``` 

   <details><summary>Show Explanation</summary>
       * `NXF_HOME` points to the directory where Nextflow internal files and additional configs are stored. The default location is your home directory. However, it might be that your home directory is not shared among all worker nodes and is only available on the master node.  In this example the variable points to your current working directory (`$PWD/.nextflow`).
      
       * `-work-dir` points in this example to your current working directory and should point to a directory that is shared between all worker nodes.

       * `-profile` defines the execution profile that should be used (local or cluster computing).

       * `-entry` is the entrypoint of the Toolkit.

       * `-params-file` sets the parameters file which defines the parameters for all tools.

       * `--databases` is the directory on the worker node where all databases are saved. Already downloaded and extracted databases on a shared file system can be configured in the database setting of the corresponding database section in the          configuration file.

       * `--output` is the output directory where all results are saved. If you want to know more about which outputs are created, then please refer to the modules section.

       * `--input.SRA.S3.path` is the path to a TSV file that lists the datasets that should be processed. Besides paired-end data there are also other input types. Please check the input section.  
   </details>

3. (Optional) You could open a second terminal in Theia to check the progress using **squeue** and **watch**.
   ```
   watch 'squeue --format="%.18i %.90j %t %.6C %m %.10M %N"'
   ```
   Watch automatically gives you the output of a command at regular intervals. The default is 2 seconds.
   You can leave the output of **watch** by pressing `CTRL + C`


### 5.5 Inspect the Toolkit results

   The following is just an example analysis. If the commands below do not output anything then it means that the pipeline is still running.
   In that case, please wait a few more minutes. 

   1. Let`s first check the size of the assemblies:
   ```
   cat output/*/*/assembly/*/megahit/*_contigs_stats.tsv | column -s$'\t' -t  
   ```

   2. The N50 in the previous output gives you a first estimation of the contig length. While the contig length looks good
   lets also check the number of bins. 
   
   ```
   ls -1 output/*/*/binning/*/metabat/
   ```

   Once the Toolkit has processed the dataset SRR492065 you should see at least the following output: 
   ```
output/SRR492065/1/binning/0.5.0/metabat/:
SRR492065_bin.1.fa
SRR492065_bin.2.fa
SRR492065_bin.3.fa
SRR492065_bin.4.fa
SRR492065_bin_contig_mapping.tsv
SRR492065_bins_stats.tsv
SRR492065_contigs_depth.tsv
SRR492065_notBinned.fa
   ```

   3. Now we want to know the taxonomy of our MAGs to check if we have found the genomes of interest.
   ```
   cat output/*/*/magAttributes/*/*/*_gtdbtk_generated_combined.tsv | cut -f 1,5
   ```
   If the dataset SRR492065 is already processed your output will contain at least the following output:
   ```
BIN_ID  classification
SRR492065_bin.1.fa      d__Bacteria;p__Bacillota;c__Bacilli;o__Lactobacillales;f__Enterococcaceae;g__Enterococcus;s__Enterococcus faecalis
SRR492065_bin.2.fa      d__Bacteria;p__Bacillota;c__Clostridia;o__Tissierellales;f__Peptoniphilaceae;g__Peptoniphilus_A;s__Peptoniphilus_A lacydonensis
SRR492065_bin.3.fa      d__Bacteria;p__Actinomycetota;c__Actinomycetes;o__Propionibacteriales;f__Propionibacteriaceae;g__Cutibacterium;s__Cutibacterium avidum
SRR492065_bin.4.fa      d__Bacteria;p__Bacillota;c__Bacilli;o__Staphylococcales;f__Staphylococcaceae;g__Staphylococcus;s__Staphylococcus aureus
   ```
   We could indeed detect in one dataset a **Staphylococcus aureus** strain.

   4. The next step could be to check the gene prediction and gene annotation. For example you could investigate the resistance genes of Staphylococcus aureus MAG. 
   In my example output I have to check the genes of a MAG with the id **SRR492065_bin.4**. 
   You could for example search for all genes that contain "antiobotic" in its name for further analysis.
   ```
   grep antibiotic output/*/*/annotation/*/rgi/SRR492065_bin.4.fa.rgi.tsv
   ```  
   At this step the tutorial ends but you could of course further investigate the detected MAGs. The Toolkit offers various additional analysis steps that can
   be added to the configuration file: 
   For example:

    * Check the completeness or contamination of your MAGs using [Checkm](https://metagenomics.github.io/metagenomics-tk/latest/modules/magAttributes/).

    * Investigate possible [plasmids](https://metagenomics.github.io/metagenomics-tk/latest/modules/plasmids/).  

    * Compare the detected MAGs using the [dereplication](https://metagenomics.github.io/metagenomics-tk/latest/modules/dereplication/) module.

Back to [Section 5 (Part 1)](part51.md)
