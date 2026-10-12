# 2️⃣3️⃣ TAR

To create a tar archive (tarball), the basic command is: `tar -cvf file.tar directory_or_files`
Where: -c = create, -v = verbose (optional, shows files), -f = filename
#### Examples
**Tar a directory** `tar -cvf backup.tar myfolder/`
**Tar multiple files** `tar -cvf config.tar file1 file2 file3`
**With compression**
**gzip (.tar.gz / .tgz)**: `tar -czvf backup.tar.gz myfolder/` -z = gzip compression
**bzip2 (.tar.bz2):** `tar -cjvf backup.tar.bz2 myfolder/` -j = bzip2 compression
**xz (.tar.xz):** `tar -cJvf backup.tar.xz myfolder/` -J = xz compression (best compression)
**Extracting** `tar -xvf backup.tar`
**or compressed:** `tar -xzvf backup.tar.gz`
