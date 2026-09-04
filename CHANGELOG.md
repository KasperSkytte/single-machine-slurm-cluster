# Changelog

## [1.2.0](https://github.com/KasperSkytte/single-machine-slurm-cluster/compare/slurm-ansible-v1.1.0...slurm-ansible-v1.2.0) (2026-09-04)


### Features

* build and install Slurm's contrib tools ([6c806fa](https://github.com/KasperSkytte/single-machine-slurm-cluster/commit/6c806fa0215ae262b9fadf1a422a266b84daed72))
* make slurm-mail optional ([0c81657](https://github.com/KasperSkytte/single-machine-slurm-cluster/commit/0c8165773c6f54d3ed95e407e90efd8f1d688f87))

## [1.1.0](https://github.com/KasperSkytte/single-machine-slurm-cluster/compare/slurm-ansible-v1.0.0...slurm-ansible-v1.1.0) (2026-07-09)


### Features

* auto-detect and configure NVIDIA GPUs for Slurm scheduling ([a7214ea](https://github.com/KasperSkytte/single-machine-slurm-cluster/commit/a7214ea2b66f60488bf715f0957cb11b5f8f3d76))
* include GPU model in slurm.conf's Gres= line ([f024ab2](https://github.com/KasperSkytte/single-machine-slurm-cluster/commit/f024ab237014bf4eb712bea5d62a5d9af745abc9))

## 1.0.0 (2026-07-09)


### Features

* initial Ansible playbook for single-node Slurm cluster ([39d03f9](https://github.com/KasperSkytte/single-machine-slurm-cluster/commit/39d03f9094e53559b7516debe3c56b8bc59e8f4b))


### Bug Fixes

* keep bashrc's squeue/sinfo env in sync with profile.d instead of duplicating it ([dd903fe](https://github.com/KasperSkytte/single-machine-slurm-cluster/commit/dd903fea1d91017e2c35e8567a6501b87aafe22a))
