# HPC di AWS
Mencoba High Performance Computing di AWS

## Login ke AWS Console
Silahkan login ke AWS Console, jika anda menggunakan akun AWS di event workshop AWS, silahkan ikuti tata cara untuk masuk ke dalam console dan region yang digunakan

## Membuat EC2 Keypair
Keypair ini akan digunakan untuk membuat HPC cluster, masuk ke menu EC2 dan buat serta download keypair tersebut. 

## Membuat Environment Cloud9
Kita akan menggunakan Cloud9 sebagai IDE dalam workshop ini, silahkan create Cloud9 environment dengan semua setting menggunakan nilai default

Upload Keypair yang sudah kita buat kedalam Cloud9

## Instalasi ParallelCluster
Dalam workshop ini, semua perintah akan dijalankan di Cloud9.

1. Install ParallelCluster
```
python3 -m pip install "aws-parallelcluster" --user 
```

2. Konfigurasi ParallelCluster

Eksekusi perintah berikut untuk melakukan konfigurasi ParallelCluster dan menyimpan hasilnya pada file wrf.yaml

```
pcluster configure --config wrf.yaml   
```

Pilih nilai berikut untuk setting konfigurasi:

- AWS Region ID [us-east-1]:  **us-east-1**  ## Atau region lain yang dianjurkan dalam workshop 
- EC2 Key Pair Name [ee-default-keypair]: **pilih keypair yang sudah dibuat**
- Scheduler [slurm]: **slurm**
- Operating System [alinux2]: **alinux2**
- Head node instance type [t2.micro]: **c5.4xlarge**   
- Number of queues [1]: **1**
- Name of queue 1 [queue1]: **queue1**                           
- Number of compute resources for queue1 [1]: **1**
- Compute instance type for compute resource 1 in queue1 [t2.micro]: **c5n.18xlarge**
- Maximum instance count [10]: **2**
- Automate VPC creation? (y/n) [n]: **y**
- Availability Zone [us-east-1a]: **us-east-1a**
- Network Configuration [Head node in a public subnet and compute fleet in a private subnet]:  **Head node in a public subnet and compute fleet in a private subnet**      

3. Tunggu proses konfigurasi tersebut sampai selesai

4. Buat file baru, ganti nilai subnet dengan referensi dari file wrf.yaml, simpan file baru ini dengan nama wrf.yaml juga

```
Region: us-east-1
Image:
  Os: alinux2
HeadNode:
  InstanceType: c5.4xlarge
  LocalStorage:
    RootVolume:
      Size: 50
  Networking:
    SubnetId: subnet-01c69a1cb858835cb     ## Public subnet yang sudah dibuat, lihat di file wrf.yaml
  DisableSimultaneousMultithreading: true
  Ssh:
    KeyName: wrf                  ## keypair yang sudah dibuat
Scheduling:
  Scheduler: slurm
  SlurmQueues:
  - Name: queue1
    ComputeResources:
    - Name: c5n18xlarge
      DisableSimultaneousMultithreading: true
      Efa:
        Enabled: true
        GdrSupport: false
      InstanceType: c5n.18xlarge
      MinCount: 0
      MaxCount: 2
    ComputeSettings:
      LocalStorage:
        EphemeralVolume:
          MountDir: /local/ephemeral
        RootVolume:
          Size: 50
    Networking:
      PlacementGroup:
        Enabled: true
      SubnetIds:
      - subnet-038e354ac0d7e115f                 ## Private subnet yang sudah dibuat, lihat di file wrf.yaml
SharedStorage:
  - MountDir: /shared      
    Name: ebs
    StorageType: Ebs
    EbsSettings:
      VolumeType: gp3
      Iops: 3000
      Size: 1024
  - MountDir: /scratch 
    Name: fsx
    StorageType: FsxLustre
    FsxLustreSettings:
      StorageCapacity: 1200            
      DeploymentType: SCRATCH_2
```


## Membuat HPC Cluster

1. Dengan file yaml tersebut, kita akan membuat cluster yang bernama "wrf-cluster" 

```
pcluster create-cluster --cluster-name wrf-cluster --cluster-configuration ./wrf.yaml 
```

2. Tunggu sampai statusnya menjadu 'CREATE_COMPLETE' dengan eksekusi perintah berikut:
```
pcluster list-clusters
```

3. Login kedalam cluster dengan perintah berikut:
```
pcluster ssh --cluster-name wrf-cluster -i ./wrf.pem
```

4. Validasi cluster dengan eksekusi perintah `sinfo` dan  `df -h` 


## Mencoba HPC Cluster

1. Buat program MPI Hello World dan jalankan di *Head Node* dengan copy dan paste kode berikut

```
mkdir -p ~/src ~/bin
cat > ~/src/hello.c << \EOF
#include <mpi.h>
#include <stdio.h>

int main(int argc, char** argv) {
    // Initialize the MPI environment
    MPI_Init(NULL, NULL);

    // Get the number of processes
    int world_size;
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    // Get the rank of the process
    int world_rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);

    // Get the name of the processor
    char processor_name[MPI_MAX_PROCESSOR_NAME];
    int name_len;
    MPI_Get_processor_name(processor_name, &name_len);

    // Print off a hello world message
    printf("Hello world from processor %s, rank %d out of %d processors\n",
           processor_name, world_rank, world_size);

    // Finalize the MPI environment.
    MPI_Finalize();
}
EOF
mpicc -o ~/bin/hello ~/src/hello.c
mpirun -n 4 ${HOME}/bin/hello 
```

2. Jalankan kode MPI tersebut di *compute node* dengan perintah berikut:
```
srun --mpi=pmix -n 4 ${HOME}/bin/hello
```
