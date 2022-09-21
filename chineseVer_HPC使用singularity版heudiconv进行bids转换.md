# 【linux使用】HPC使用singularity版heudiconv进行bids转换

@author: Wanru Li  
@contact: wliwanru@gmail.com
@create: 2022/9/20

# Part1: docker image 转化为 singularity image

## toolbox: docker2singularity
https://quay.io/repository/singularity/docker2singularity

### docker2singularity安装
``` bash
docker run -v /var/run/docker.sock:/var/run/docker.sock -v /media/wanru/HDD11/software/singularity:/output --privileged -t --rm quay.io/singularity/docker2singularity ubuntu:22.02

```

output like: 
``` bash
wanru@wanru-ubuntu:/media/wanru/HDD11/software/singularity$ docker run -v /var/run/docker.sock:/var/run/docker.sock -v /media/wanru/HDD11/software/singularity:/output --privileged -t --rm quay.io/singularity/docker2singularity ubuntu:22.02
Unable to find image 'quay.io/singularity/docker2singularity:latest' locally
latest: Pulling from singularity/docker2singularity
9d48c3bd43c5: Pull complete 
7f94eaf8af20: Pull complete 
9fe9984849c1: Pull complete 
e17f0ad88222: Pull complete 
e8bd20cf6e8a: Pull complete 
dbbc603b9086: Pull complete 
17443955f0b9: Pull complete 
200e01a38690: Pull complete 
568c9685b811: Pull complete 
d4426c72f40f: Pull complete 
797f61e0aa0a: Pull complete 
470759a1ccf3: Pull complete 
Digest: sha256:377986628b009f7f6ec60250db3df24dc2b32572fe1534a4cba4f2a29e7620b0
Status: Downloaded newer image for quay.io/singularity/docker2singularity:latest

Image Format: squashfs
Docker Image: ubuntu:22.02

Unable to find image 'ubuntu:22.02' locally
docker: Error response from daemon: manifest for ubuntu:22.02 not found: manifest unknown: manifest unknown.
See 'docker run --help'.

```

## 将heudiconv的docker image转换为singularity image
check docker images
``` bash
wanru@wanru-ubuntu:/media/wanru/HDD11/software/singularity$ docker images
REPOSITORY                               TAG       IMAGE ID       CREATED         SIZE
quay.io/singularity/docker2singularity   latest    64bc3184862f   8 weeks ago     382MB
bids/validator                           latest    011ad9c7a621   3 months ago    635MB
nipy/heudiconv                           latest    c0161baff9e2   4 months ago    1.38GB
```
进行转换
``` bash
docker run -v /var/run/docker.sock:/var/run/docker.sock -v /media/wanru/HDD11/software/singularity:/output --privileged -t --rm quay.io/singularity/docker2singularity nipy/heudiconv
```

output like: 
``` bash
wanru@wanru-ubuntu:/media/wanru/HDD11/software/singularity$ docker run -v /var/run/docker.sock:/var/run/docker.sock -v /media/wanru/HDD11/software/singularity:/output --privileged -t --rm quay.io/singularity/docker2singularity nipy/heudiconv

Image Format: squashfs
Docker Image: nipy/heudiconv

Inspected Size: 1378 MB

(1/10) Creating a build sandbox...
(2/10) Exporting filesystem...
(3/10) Creating labels...
(4/10) Adding run script...
(5/10) Setting ENV variables...
(6/10) Adding mount points...
(7/10) Fixing permissions...
(8/10) Stopping and removing the container...
(9/10) Building squashfs container...
INFO:    Starting build...
INFO:    Creating SIF file...
INFO:    Build complete: /tmp/nipy_heudiconv-2022-05-12-91a4bf56c041.sif
(10/10) Moving the image to the output folder...
    462,348,288 100%   55.14MB/s    0:00:07 (xfr#1, to-chk=0/1)
Final Size: 441MB
```


# Part2: 使用singularity版heudiconv进行dicom to bids转换
## 上传singularity heudiconv到HPC

在HPC上，我把转化好的image文件放到了如下路径：
``` 
$HOME/software/conv2bids/nipy_heudiconv-2022-05-12-91a4bf56c041.sif
```

## 在HPC上进行转换

这里的test-organized_dicom文件夹里放着subject022 session01的dicom数据
需要事先创建好raw_bids, raw_bids/code/heuristic.py, 详见 heudiconv使用指南_v1.0.pdf 

``` text
── test
    ├── organized_dicom
    │   └── 022
    │       └── 01
    │           ├── 1003-t1_mprage_sag_iso_mww64CH
    │           ├── 1004-gre_field_mapping_TR1000_2d5mm_mag-4.92
    │           ├── 1004-gre_field_mapping_TR1000_2d5mm_mag-7.38
    │           ├── 1005-gre_field_mapping_TR1000_2d5mm_pha-7.38
    │           ├── 1006-sms4_bold_TR1000_2d5mm_re1
    │           ├── 1007-sms4_bold_TR1000_2d5mm_re2
    │           ├── 1008-sms4_bold_TR1000_2d5mm_re3
    │           ├── 1009-sms4_bold_TR1000_2d5mm_re4
    │           ├── 1010-sms4_bold_TR1000_2d5mm_re5
    │           ├── 1012-sms4_bold_TR1000_2d5mm_re6
    │           ├── 1013-sms4_bold_TR1000_2d5mm_loc1
    │           └── 1014-sms4_bold_TR1000_2d5mm_loc2
    ├── raw_bids
    │   ├── code
    │   │   └── heuristic.py
    └── raw_dicom
```


## 转换代码

注释部分的使用详见 heudiconv使用指南_v1.0.pdf

``` bash
#!/bin/bash

module purge
module load singularity

HOME=/your/HPC/home/path  
converter_dir=${HOME}/software/conv2bids
sif=${converter_dir}/nipy_heudiconv-2022-05-12-91a4bf56c041.sif
base_dir=${HOME}/data/project_data/test


unset PYTHONPATH

# subject=022
# session=01

for subject in 022 021
do
	for session in 01
	do

		## ------- this is for generating dicom info and heuristic.py file ------
		singularity run --cleanenv -B $base_dir:/base $sif \
		-d /base/organized_dicom/{subject}/{session}/*/*.dcm \
		-o /base/raw_bids/ \
		-f convertall \
		-s $subject -ss $session \
		-c none \
		--overwrite
		
		
		## ------------------ run converter --------------------
		singularity run --cleanenv -B $base_dir:/base $sif \
		-d /base/organized_dicom/{subject}/{session}/*/*.dcm \
		-o /base/raw_bids/ \
		-f /base/raw_bids/code/heuristic.py \
		-s $subject -ss $session \
		-c dcm2niix \
		-b \
		--overwrite
	done
done

```

对比docker版heudiconv，可以看到使用上与docker基本没区别

``` bash
docker run -i --rm -v {base_dir}:/base nipy/heudiconv:latest 
-d /base/dicom/{subject}/{session}/*/*.dcm 
-o /base/bids/ "
-f /base/bids/code/heuristic.py 
-s {i_subject} -ss {i_session} 
-c dcm2niix \
-b \
--overwrite 
```

