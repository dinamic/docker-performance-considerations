# Running composer in docker - performance considerations

## Overview

I wanted to see what's the performance penalty of running composer in a docker container and whether that's related to CPU operations or access to the filesystem. I wanted to rule out filesystem as a bottleneck, so I ran a few tests with different scenarios that I can think about.

Overall, the fastest one is using a docker non-shared folder. Second place goes to using a mount bound to a non-shared folder, which just about aliviates the performance penalty. Third place goes to native performance on macOS as well as using a symlink to a non-shared folder.

It is quite odd the first place is won by a big margin - nearly half the time of running composer natively on macOS.

## Tests conducted, ordered by performance

### Using non-shared folder
```
# within container
cd /root
cp /var/www/composer.* .
time composer install --prefer-dist

real	0m 13.92s
user	0m 7.55s
sys	0m 3.82s
```

I thought that maybe I can copy composer.json and composer.lock files to another folder, install the dependencies and then copy them back. Obviously the performance penalty is not worth it, also any scripts that hook into composer events would not work as intended.

```
# within container
time mv -fi /root/vendor/ ./

real	2m 54.52s
user	0m 2.03s
sys	0m 15.12s
```

### Using shared folder with vendor bound to a non-shared folder

To use mount bounds, we need to have the container privileged. We should only be doing that to containers we trust, but this one runs our own software, so it should be good to go.

```
cd /var/www
rm -rf vendor
mkdir /root/vendor
mount --bind /root/vendor vendor
time composer install --prefer-dist --no-scripts

real	0m 15.64s
user	0m 7.19s
sys	0m 4.25s
```

### native on macOS:
```
# native, on macOS
time composer install --prefer-dist --no-scripts

real	0m24.292s
user	0m7.851s
sys	0m12.596s
```

### Native, using ram disk
@see https://devops.stackexchange.com/questions/4012/does-docker-on-macos-support-tmpfs

This should have been a lot faster. Maybe a bottleneck with composer cache - it sits in ~/.composer/cache, which is on the SSD drive.

```
time composer install --prefer-dist --no-scripts

real	0m28.942s
user	0m9.218s
sys	0m10.654s
```

### Using shared folder with vendor symlink to non-shared folder
```
# within container
cd /var/www
rm -rf vendor && ln -s /root/vendor ./vendor
mkdir /root/vendor
time composer install --prefer-dist

real	0m 26.47s
user	0m 7.81s
sys	0m 5.16s
```

### Using shared folder with cached strategy
```
# within container
rm -rf vendor
time composer install --prefer-dist

real	2m 29.82s
user	0m 11.52s
sys	0m 11.73s
```

### Using shared folder with delegated strategy
```
# within container
rm -rf vendor
time composer install --prefer-dist --no-scripts

real	2m 36.45s
user	0m 11.89s
sys	0m 11.25s
```

### Using shared folder with delegated strategy on ramdisk

```
# within container
rm -rf vendor
time composer install --prefer-dist --no-scripts

real	3m 37.19s
user	0m 15.74s
sys	0m 16.75s
```

Using a symlink of /root/.composer/cache to /var/www/.composer_cache.

```
real	2m 15.58s
user	0m 10.93s
sys	0m 11.43s
```
