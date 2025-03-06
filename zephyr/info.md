# Various info regarding Zephyr RTOS and west

## Configuring and building a project
Most trivial case:
```
west build --build-dir build . --pristine --board <board_name>
```

`<board_name>` must be present in `<zephyr>/boards/`
By default is nothing is specified a `prj.conf` is the configuration file name.
Can be changed with `-DCONF_FILE=<conf file>` option.

Any option that is not recognized by the `west` seems to be forwarded to a `cmake`.
So if you want to provide, say, a toolchain file it can be done as:
```
west build --build-dir build . --pristine --board <board_name> --toolchain /path/to/toolchain.cmake
```

To simply build already configured:
```
west build
```
`west` assumes `build` as a building directory

To build a specific target:
```
west build -t <target name>
```

## How to flash
As easy as:
```
west flash
```

## Enter a configuration terminal menu:
```
west build -t menuconfig
```

## Get partitions info
```
west build -t partition_manager_report
```
***Note***: works only with `sysbuild` (default, if no `--no-sysbuild` option is specified)

