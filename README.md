# Docker Registry Shunter


This repository provides the `docker-shunt` and `drs` commands which can be used to migrate image tags from
one Docker registry to another. If you're trying to move a single image tag you probably want to call `docker-shunt`
directly. If you want to move lots of images at once, you'll want to use `drs` which supports tag pattern matching,
tag sorting, parallelized workers, exclusion rules, etc. 

**shunt**
>push or pull (a train or part of a train) from the main line to a siding or from one track to another.

See also: [Thomas & Friends](https://www.youtube.com/watch?v=vs9V5njC9X0)

## Compatibility 

This script was written on and meant to be run on Linux hosts. It will likely work on other platforms too, but there are 
some known issues with `drs`:

  - The default tag sorting scheme used by `drs` is `sort -Vr` the `V` flag allows for semantic version sorting but is only available in gnusort and is not supported by OS X or Windows native sort. You can either change the sort arguments using the `SORT_ARGS` environment variable (default: `-Vr`) or install a compatible sorting program from coreutils ([OS X via Brew](https://apple.stackexchange.com/questions/69223/how-to-replace-mac-os-x-utilities-with-gnu-core-utilities), [Windows](http://gnuwin32.sourceforge.net/packages/coreutils.htm)).
  - This script is meant to be used when moving between self-hosted private registries, shunting from docker hub may not work due to API inconsistencies **TODO: Maybe fix this** 

## Quick Start

1. Ensure Docker is installed
1. Clone this repository or download the script files directly.
1. Make sure both files are executable `chmod +x drs docker-shunt`
1. Put them either: in your PATH or next to each other anywhere

## `drs`
This script functions as the "yardmaster" in train terms; determining which image tags should be migrated and how
many to do at one time. In order for this script to function you must provide source and destination registries:

```
drs -s docker.yourregistry.com -d docker.yourotherregistry.com
```

This script will query the `v2/_catalog` endpoint and loop through all the images it finds and query the tags for those
images. It will then "shunt" the image tags from the source registry to the destination registry by calling `docker-shunt`
in the background.

The `drs` command is VERY customizable, both via flags and environment variables.

### `drs` environment configuration

It is assumed that most users will not have to regularly modify these variables.

  - `REGISTRY_PROTOCOL` (`https`) - Used by `drs` when querying the catalog endpoint. If for some reason your registry isn't using TLS this will allow `drs` to still list the catalog and tags.
  - `DRY_RUN` (`false`) - Used to run the script but only output what would happen. Doesn't actually call `docker-shunt` or move anything.
  - `MAX_CATALOG_SIZE` (`1000`) - The catalog endpoint is paginated, but `drs` is complex enough, so instead of using catalog pagination it tries to get everything at once. **TODO: FIX THIS TO USE PAGINATION**
  - `IMAGE_PATTERN` (`".*"`) - This is the pattern `drs` uses when listing images to determine what to shunt from one registry to the next. By default `drs` will try to move every image it finds (but not every tag).
  - `TAG_PATTERN` (`"v[0-9].*"`) - This is the pattern `drs` uses when listing tags to determine what should be migrated. We assume tags are of the form `v1.0.0` or `v1-0-0` etc. If your versioning scheme is different, or you want to grab only "latest" tags or something, change this value.
  - `SORT_ARGS` (`-Vr`) - This is the sorting paradigm used on tags. The default value uses `V` to semantically sort version numbers and `r` to make sure they're reversed, meaning the most recent versions are favored.

### `drs` flag configuration

These values are expected to be changed pretty frequently, so they're flags.

**Required:**

  - `-s` the source registry, eg:`docker.yourdomain.com`
  - `-d` the destination registry, eg: `otherdocker.yourdomain.com`
  
**Optional:**

  - `-h` Help. Outputs usage info
  - `-n` (`5`) Maximum number of tags (matching `TAG_PATTERN`) to move for each image.
  - `-w` (`10`) Number of backgrounded `docker-shunt` calls to have running at once.
  - `-e` (`/dev/null`) Path to a newline delimited file of `image:tag` strings that should be skipped even if they match `IMAGE_PATTERN` and `TAG_PATTERN`.
  - `-l` (`drs-log-DATETIME`) Path to the logfile that `drs` and its calls to `docker-shunt` should log to. Note that because `docker-shunt` calls are backgrounded order of events in the log file is not necessarily significant.
  - `-c` (`drs-completed-DATETIME`) Path to a file that will have an `image:tag` string appended to it after every successful `docker-shunt` call.

## `docker-shunt`

This is a utility command that pulls down a given image tag from a source registry,  tags it for the destination 
registry, and then pushes it to the destination registry. `drs` uses this to do the actual shunting between registries 
but it can also be called on its own. It is configured via flags.

```
docker-shunt -s docker.yourdomain.com -d otherdocker.yourdomain.com -i your-image -t latest
```

### `docker-shunt` flag configuration

**Required:**

  - `-s` the source registry, eg:`docker.yourdomain.com`
  - `-d` the destination registry, eg: `otherdocker.yourdomain.com`
  - `-i` the image name
  - `-t` the tag name
  
**Optional:**

  - `-h` Help. Outputs usage info
  - `-l` (`/dev/stdout`) Path to the logfile where debug information should be sent.
  - `-c` (`/dev/null`) Path to a file that will have an `image:tag` string appended to it after every successful call.
