Using this command to setup cmake

```
cmake ../server/ -DCMAKE_BUILD_TYPE=Debug -DCMAKE_EXPORT_COMPILE_COMM
ANDS=ON -DPLUGIN_MROONGA=NO
```

It adds a compile_commands.json file to the build directory, which clangd can pick up (even if the build directory is beside the source directory!)
