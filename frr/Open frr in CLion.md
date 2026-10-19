
# CLion pre-configuring failed
FRR doesn't use CMake. It uses autotools (`bootstrap.sh` → `configure` → `make`), so CLion has to treat it as a Makefile project. A fresh clone has no `configure` script and no `Makefile` yet, which is why CLion's pre-configuring step usually fails. Missing build dependencies are the other common cause.

**1. Run the build once in a terminal to see the real error**

```bash
cd frr
./bootstrap.sh
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var \
  --enable-dev-build --disable-doc
make -j$(nproc)
```

If `configure` stops, it will name the missing package. CLion often hides that message.

**2. Install the dependencies** (Ubuntu/Debian example)

```bash
sudo apt install git autoconf automake libtool make libreadline-dev texinfo \
  pkg-config libpam0g-dev libjson-c-dev bison flex libc-ares-dev python3-dev \
  python3-sphinx install-info build-essential libsnmp-dev perl libcap-dev \
  libelf-dev libunwind-dev protobuf-c-compiler libprotobuf-c-dev libpcre2-dev
```

FRR also needs **libyang** in a recent 2.x version. A distro package may be too old; if `configure` complains about libyang, build it from source by following FRR's docs for your distro and FRR version.

**3. Tell CLion how to configure it**

Go to Settings → Build, Execution, Deployment → Makefile and set:

- **Pre-configuration commands:** `./bootstrap.sh && ./configure --enable-dev-build --disable-doc`
- **Build target:** `all`

Then run Tools → Makefile → Clean and Reload Makefile Project. If you already ran `configure` yourself in step 1, you can leave the pre-configuration field empty, and CLion will use the existing `Makefile`.

**Alternative that often works better: a compilation database**

```bash
sudo apt install bear
make clean && bear -- make -j$(nproc)
```

This creates `compile_commands.json`. Close the project in CLion, then use File → Open on that file and choose "Open as Project." Code navigation and indexing then work without CLion running the autotools steps.

**If you're on macOS or Windows:** FRR is Linux-focused. Use a Linux remote toolchain, WSL, or a Docker toolchain in CLion (Settings → Toolchains) rather than building natively.

If it still fails, paste the error from CLion's Build/Makefile tool window or from `./configure` and I'll pinpoint it.

# Libyang version is too old in Debian 12.

Your distro's packaged libyang is too old: it has 2.1.30, and FRR needs at least 2.1.128. Don't install those packages. Build libyang from source instead.

**1. Remove the old dev package, if you installed it**

```bash
sudo apt remove libyang2-dev
```

Having two versions installed can cause FRR to pick up the wrong headers.

**2. Build and install libyang**

```bash
sudo apt install cmake libpcre2-dev build-essential git

cd /opt/code
git clone https://github.com/CESNET/libyang.git
cd libyang
git checkout v2.1.128
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX:PATH=/usr -DCMAKE_BUILD_TYPE:String="Release" ..
make -j$(nproc)
sudo make install
sudo ldconfig
```

**3. Check that it's found, then configure FRR again**

```bash
pkg-config --modversion libyang   # should print 2.1.128
cd /opt/code/frr
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var \
  --enable-dev-build --disable-doc
```

If your FRR checkout is very new and later complains that it needs libyang 3.x, check out a newer tag instead (`git tag` lists them) and rebuild. The `doc/developer/building-libyang.rst` file in your FRR tree names the exact version that checkout expects.

Once `configure` succeeds, go back to CLion and run Clean and Reload Makefile Project.

# After configure succeeds, CLion still has issue
This error comes from a conflict between your terminal build and CLion's.

CLion's default Autotools template runs `configure` out-of-tree: it runs from a separate build folder and points back at the source with `..`. You already ran `configure` inside `/opt/code/frr` itself, and autotools refuses to do an out-of-tree configure when the source folder is already configured. The message says exactly that: "source directory already configured; run make distclean there first."

Pick one of these two fixes.

**Option A (simplest): let CLion use the build you already configured**

1. Open Settings → Build, Execution, Deployment → Makefile.
2. Delete everything in **Pre-configuration commands** so the field is empty.
3. Set **Build directory** to `/opt/code/frr`, the project root. You can also leave it empty if your CLion version uses the root by default.
4. Run Tools → Makefile → Clean and Reload Makefile Project.

CLion will now read the `Makefile` your terminal `configure` produced, with your libyang and other options already applied.

**Option B: let CLion manage the build out-of-tree**

1. Clean the in-tree configuration:
    
    ```bash
    cd /opt/code/frrmake distclean
    ```
    
2. In the Makefile settings, replace the pre-configuration commands with:
    
    ```sh
    "${PROJECT_DIR:-..}/configure" --prefix=/usr --sysconfdir=/etc --localstatedir=/var --enable-dev-build --disable-doc
    ```
    
    `autoreconf` already ran, so you don't need it again. Without your flags, CLion would run a bare `configure`.
3. Run Clean and Reload again.

Option A is less hassle. Choose B only if you want to keep the source tree free of build output.