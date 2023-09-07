---
title: "How Ubuntu repository works"
summary: Understanding how ubuntu apt repositories work
date: 2023-09-05
weight: 1
tags: ["ubuntu","linux"]
author: "Clement Thomas"
---

I always never understood what those multiple keywords (`stable`, `main`, `contrib`, `non-free` etc.) that come after the url in an apt repository file really mean. So thought will explore what that really means. Though lot of what we discuss apply to any debian based systems, the following commands were tested on ubuntu 22.04 (jammy)

### Bit of theory
A repository or a package archive is used to store and retrieve packages when required. A repository is configured with a `.list` file under `/etc/apt/sources.list.d` which contains the URL and few other information on how to check available packages and retrieve them for installation. The format is

```
deb uri distribution [component1] [component2] [...]
```

and an example

```
deb https://deb.debian.org/debian stable main contrib non-free
```

Here `deb` means this repository is serving binary packages and `https://deb.debian.org/debian` is the root of the archive where the repository exists. From now on we will use `$ARCHIVE_ROOT` to denote the root of the archive. The distribution (`stable` in the example) specifies a subdirectory in `$ARCHIVE_ROOT/dists`, so `$ARCHIVE_ROOT/dists/stable`. distribution typically corresponds to Suite (is usually a single word and in debian world this used to be one of oldstable, stable, testing, unstable or experimental with optional suffixes such as -updates ) or Codename (focal, jammy etc.)

apt will look for an `InRelease` or `Release` file from the `$ARCHIVE_ROOT/dists/stable` directory. InRelease files are signed in-line while Release files should have an accompanying Release.gpg file. The Release file lists the index files `Packages` or `Packages.gz` for the distribution and their hashes. 

There can be multiple components under a distribution (in our example we have `main`,`contrib` and `non-free`) and each will be a directory under `$ARCHIVE_ROOT/dists/stable`. For eg. `$ARCHIVE_ROOT/dists/stable/main`, `$ARCHIVE_ROOT/dists/stable/contrib` etc. To download the index of the main component, apt would scan the Release file in the distribution directory ie. `$ARCHIVE_ROOT/dists/stable/InRelease` for where to find the `Packages.gz` of each component and what packages are available in each component. Binary package indices are in `binary-$arch` subdirectory of the component directories. so say we have a binary deb file `app_1.5.6_amd64.deb` that we intend to serve under `main` component, this deb can be put under `$ARCHIVE_ROOT/dists/stable/main/binary-amd64/app_1.5.6_amd64.deb`.

Ok enough theory. Lets check this out. 

### Quick Demo setup

In this demo setup, we will be setting up a ubuntu repository with `jammy` distribution and two components (`main` and `obsolete`) with each component having a package. We will also sign them with gpg keys and see how we can configure a client host to retrive the packages from our repository. 

Let us install the required packages to sign and also create required repository meta data files. Also let us create the required directory structure for our repo.

```
sudo apt-get install dpkg-dev dpkg-sig

mkdir repo-root/dists/jammy/main/binary-amd64 -p
mkdir repo-root/dists/jammy/obsolete/binary-amd64 -p

tree repo-root/
repo-root/
└── dists
    └── jammy
        ├── main
        │   └── binary-amd64
        └── obsolete
            └── binary-amd64

```

Lets us use [fpm](https://fpm.readthedocs.io/en/latest/packages/deb.html) to create two deb packages from zip files. Easy way to create debs 🙂 Lets create a deb of terraform version `0.1.0` and `1.5.6`.  and move the debs inside the `binary-amd64` directory. 

```
wget https://releases.hashicorp.com/terraform/1.5.6/terraform_1.5.6_linux_amd64.zip
fpm -s zip -t deb --prefix /usr/bin -n terraform -v 1.5.6 terraform_1.5.6_linux_amd64.zip

wget https://releases.hashicorp.com/terraform/0.1.0/terraform_0.1.0_linux_amd64.zip
fpm -s zip -t deb --prefix /usr/bin -n terraform -v 0.1.0 terraform_0.1.0_linux_amd64.zip

mv terraform_1.5.6_amd64.deb repo-root/dists/jammy/main/binary-amd64
mv terraform_0.1.0_amd64.deb repo-root/dists/jammy/obsolete/binary-amd64

tree repo-root/
repo-root/
└── dists
    └── jammy
        ├── main
        │   └── binary-amd64
        │       └── terraform_1.5.6_amd64.deb
        └── obsolete
            └── binary-amd64
                └── terraform_0.1.0_amd64.deb

6 directories, 2 files
```
Let us create a gpg key for email `repoadmin@example.org` to sign the packages. 

```
gpg --gen-key
...
...
...
gpg --list-keys
...
pub   rsa3072 2023-09-05 [SC] [expires: 2025-09-04]
      999D25D307E669F740DB485B4862D9E6B44008D5
uid           [ultimate] Repo Admin <repoadmin@example.org>
sub   rsa3072 2023-09-05 [E] [expires: 2025-09-04]
```

Let us export our public key and put a copy in the repo, so anyone using our repository will be able to verify it using our public key.

```
gpg --output keyFile --armor --export repoadmin@example.org
mv keyFile repo-root
```
and now the directory structure of our repo will look like
```
repo-root/
├── dists
│   └── jammy
│       ├── main
│       │   └── binary-amd64
│       │       └── terraform_1.5.6_amd64.deb
│       └── obsolete
│           └── binary-amd64
│               └── terraform_0.1.0_amd64.deb
└── keyFile

6 directories, 3 files
```

Now let us sign the debs with the gpg key

```
dpkg-sig --sign builder -k repoadmin@example.org  repo-root/dists/jammy/main/binary-amd64/terraform_1.5.6_amd64.deb 
dpkg-sig --sign builder -k repoadmin@example.org  repo-root/dists/jammy/obsolete/binary-amd64/terraform_0.1.0_amd64.deb
```

Now its time to create the package indexes for each of our component.

```
cd repo-root

dpkg-scanpackages dists/jammy/main/binary-amd64/ > dists/jammy/main/binary-amd64/Packages
gzip -c dists/jammy/main/binary-amd64/Packages > dists/jammy/main/binary-amd64/Packages.gz

dpkg-scanpackages dists/jammy/obsolete/binary-amd64/ > dists/jammy/obsolete/binary-amd64/Packages
gzip -c dists/jammy/obsolete/binary-amd64/Packages > dists/jammy/obsolete/binary-amd64/Packages.gz

tree repo-root/
repo-root/
├── dists
│   └── jammy
│       ├── main
│       │   └── binary-amd64
│       │       ├── Packages
│       │       ├── Packages.gz
│       │       └── terraform_1.5.6_amd64.deb
│       └── obsolete
│           └── binary-amd64
│               ├── Packages
│               ├── Packages.gz
│               └── terraform_0.1.0_amd64.deb
└── keyFile

6 directories, 7 files

```
The following will be content of Packages in the main component. 

```
Package: terraform
Version: 1.5.6
Architecture: amd64
Maintainer: <clement@lap01>
Installed-Size: 63672
Filename: dists/jammy/main/binary-amd64/terraform_1.5.6_amd64.deb
Size: 20491454
MD5sum: 1e78cefa67e2d8430683faa15603a177
SHA1: 59f176a004b858ea36e7581807cdaeac46448e41
SHA256: b8b290a941ce26f8031eab4f7ceaba72a3fa926be4a218ba382249b14adb7ac3
Section: default
Priority: optional
Homepage: http://example.com/no-uri-given
Description: no description given
License: unknown
Vendor: none
```

Now its time to create the Release files and  sign them

```

apt-ftparchive release -o APT::FTPArchive::Release::Codename=jammy . > Release

apt-ftparchive release . > Release; 

gpg --default-key repoadmin@example.org --clearsign -o InRelease Release; gpg --default-key repoadmin@example.org -abs -o Release.gpg Release

$tree repo-root/
repo-root/
├── dists
│   └── jammy
│       ├── InRelease
│       ├── main
│       │   └── binary-amd64
│       │       ├── Packages
│       │       ├── Packages.gz
│       │       └── terraform_1.5.6_amd64.deb
│       ├── obsolete
│       │   └── binary-amd64
│       │       ├── Packages
│       │       ├── Packages.gz
│       │       └── terraform_0.1.0_amd64.deb
│       ├── Release
│       └── Release.gpg
└── keyFile

6 directories, 10 files

```


Now the files required to serve the repository is ready and we will need to serve it over http. We will use python http.server for this.

```
python3 -m http.server --directory repo-root/ --bind 127.0.0.1 8090
```

Our server side setup is done. Now lets add a `test.list` file and verify if we are able to pull the packages from this repo

```
echo 'deb [arch=amd64] http://127.0.0.1:8090/ jammy main obsolete' > /etc/apt/sources.list.d/test.list

# lets add the gpg key from repo

wget -O - http://127.0.0.1:8090/keyFile | gpg --dearmor > /etc/apt/trusted.gpg.d/test.gpg
chmod 644 /etc/apt/trusted.gpg.d/test.gpg

root@lap01:~# apt-get update
Get:1 http://127.0.0.1:8090 jammy InRelease [3,044 B]
Get:2 http://127.0.0.1:8090 jammy/main amd64 Packages [376 B]
Get:3 http://127.0.0.1:8090 jammy/obsolete amd64 Packages [381 B]


root@lap01:/etc/apt/sources.list.d# apt-cache madison terraform                                                                                   
 terraform |    1.5.6-1 | https://apt.releases.hashicorp.com jammy/main amd64 Packages
 terraform |      1.5.6 | http://127.0.0.1:8090 jammy/main amd64 Packages             
 terraform |    0.11.15 | https://apt.releases.hashicorp.com jammy/main amd64 Packages
 terraform |    0.11.14 | https://apt.releases.hashicorp.com jammy/main amd64 Packages
 terraform |      0.1.0 | http://127.0.0.1:8090 jammy/obsolete amd64 Packages
```

We can see the terraform debs from our repository is also visible now. We can go ahead and try to install it. 

```
root@lap01:~# apt-get install terraform=0.1.0
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  terraform
  0 upgraded, 1 newly installed, 0 to remove and 15 not upgraded.
  Need to get 23.0 MB of archives.
  After this operation, 84.8 MB of additional disk space will be used.
  Get:1 http://127.0.0.1:8090 jammy/obsolete amd64 terraform amd64 0.1.0 [23.0 MB]
  Fetched 23.0 MB in 0s (193 MB/s)   
  Selecting previously unselected package terraform.
  (Reading database ... 226066 files and directories currently installed.)
  Preparing to unpack .../terraform_0.1.0_amd64.deb ...
  Unpacking terraform (0.1.0) ...
  Setting up terraform (0.1.0) ...
  root@lap01:~# terraform version
  Terraform v0.1.0
  ```

  Tada it worked. 

### References

- https://wiki.debian.org/DebianRepository/Format
- https://help.ubuntu.com/community/CreateAuthenticatedRepository

