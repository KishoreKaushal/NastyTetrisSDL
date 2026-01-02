# NastyTetrisSDL

### Setup

### Building SDL_mixer

```shell
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DMPG123_LFS=OFF -DCMAKE_PREFIX_PATH="../SDL3"
cmake --build build -- -j 14
```



```shell
git clone <repo_url>
git submodule update --init --recursive

```