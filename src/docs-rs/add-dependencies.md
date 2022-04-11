# Add a dependency to the build environment

Rustwide internally uses `rustops/crates-build-env` as the build environment for the crate. If you want to add a system package for crates to link to, this is place you're looking for.

## Getting started

First, clone the crates-build-env and the docs.rs repos:

```sh
git clone https://github.com/rust-lang/crates-build-env
git clone https://github.com/rust-lang/docs.rs
```

Set the path to the directory of your crate. This must be an absolute path, not a relative path! On platforms with coreutils, you can instead use `$(realpath ../relative/path)` (relative to the docs.rs directory).

```sh
YOUR_CRATE=/path/to/your/crate
```

## Add package 

Next, add the package to `crates-build-env/linux/packages.txt` in the correct alphabetical order. This should be the name of a package in the **Ubuntu 20.04** Repositories. See [the package home page](https://packages.ubuntu.com/) for a full list/search bar, or use `apt search` locally.

## Building the image

Now build the image. This will take a very long time, probably 10-20 minutes.

```sh
cd crates-build-env/linux
docker build --tag build-env .
```

## Testing the image

Use the image to build your crate.

```sh
cd ../../docs.rs
cp .env.sample .env
docker-compose build
# avoid docker-compose creating the volume if it doesn't exist
if [ -e "$YOUR_CRATE" ]; then
  docker-compose run -e DOCSRS_DOCKER_IMAGE=build-env \
                     -e RUST_BACKTRACE=1 \
                     -v "$YOUR_CRATE":/opt/rustwide/workdir \
    web build crate --local /opt/rustwide/workdir
else
  echo "$YOUR_CRATE does not exist";
fi
```

## Making multiple changes

If your build fails even after your changes, it will be annoying to rebuild the image from scratch just to add a single package. Instead, you can make changes directly to the Dockerfile so that the existing packages are cached. Be sure to move these new packages from the Dockerfile to `packages.txt` once you are sure they work.

On line 7 of the Dockerfile, add this line: `RUN apt-get install -y your_second_package`.
Rerun the build and start the container; it should take much less time now:

```sh
cd ../crates-build-env/linux
docker build --tag build-env .
cd ../../docs.rs
docker-compose run -e DOCS_RS_LOCAL_DOCKER_IMAGE=build-env \
                     -e RUST_BACKTRACE=1 \
                     -v "$YOUR_CRATE":/opt/rustwide/workdir \
    web build crate --local /opt/rustwide/workdir
```

## Run the lint script

Before you make a PR, run the shell script `lint.sh` and make sure it passes. It ensures `packages.txt` is in order and will tell you exactly what changes you need to make if not.

```sh
cd ../crates-build-env
./lint.sh
```

## Make a pull request

Once you are sure your package builds, you can make a pull request to get it adopted upstream for docs.rs and crater. Go to https://github.com/rust-lang/crates-build-env and click 'Fork' in the top right. Locally, add your fork as a remote in git and push your changes:

```sh
git remote add personal https://github.com/<your_username_here>/crates-build-env
git add -u
git commit -m 'add packages necessary for <your_package_here> to compile'
git push personal
```

Back on github, make a pull request:

1. Go to https://github.com/rust-lang/crates-build-env/compare
2. Click 'compare across forks'
3. Click 'head repository' -> <your_username>/crates-build-env
4. Click 'Create pull request'
5. Add a description of what packages you added and what crate they fixed
6. Click 'Create pull request' again in the bottom right.

Hopefully your changes will be merged quickly! After that you can either publish a point release (rebuilds your docs immediately) or request for a member of the docs.rs team to schedule a new build (may take a while depending on their schedules).
