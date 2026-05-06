# Wireshark RPM – Offline / Standalone Installation and Testing

This document describes how to install the Wireshark RPMs produced by the
[`Build EL RPMs`](../../.github/workflows/rpm.yml) CI workflow on a machine
that has **no Internet access**, and how to execute the functional test suite
on such a machine.

---

## 1. RPM artefacts produced by the CI

Each successful workflow run uploads three artefact bundles:

| Artefact name | Built on | Packages inside |
|---|---|---|
| `wireshark-el8-rpms` | AlmaLinux 8 | `wireshark`, `wireshark-qt`, `wireshark-devel` |
| `wireshark-el7-rpms` | CentOS 7 (devtoolset-9) | `wireshark`, `wireshark-qt`, `wireshark-devel` |
| `wireshark-el6-rpms` | CentOS 6 (devtoolset-7) | `wireshark`, `wireshark-devel` (no Qt GUI) |

Each bundle is a `.zip` archive containing the RPMs.  Download and unzip them
on the target machine (or on a machine with internet access and transfer via
USB/SCP):

```bash
unzip wireshark-el8-rpms.zip -d wireshark-el8-rpms/
```

---

## 2. EL 8 – offline installation (AlmaLinux 8 / Rocky Linux 8 / RHEL 8)

### 2.1 Runtime dependencies

The base `wireshark` RPM (tshark, dumpcap, plugins) links against packages
that are part of the default AlmaLinux/RHEL 8 distribution plus EPEL:

```
glib2           libpcap         zlib            libgcrypt
pcre2           c-ares          speexdsp        libxml2
libcap          krb5-libs       lz4-libs        snappy
libmaxminddb    gnutls          brotli          libzstd
opus
```

The `wireshark-qt` sub-package additionally requires:

```
qt5-qtbase      qt5-qtmultimedia    qt5-qtsvg
xdg-utils       hicolor-icon-theme
```

### 2.2 Pre-downloading all dependencies on a connected machine

```bash
# On a connected AlmaLinux 8 machine, download wireshark + all deps into ./rpms/
dnf install -y epel-release
/usr/bin/crb enable

mkdir -p rpms
dnf download --resolve --destdir=rpms \
    glib2 libpcap zlib libgcrypt pcre2 c-ares speexdsp libxml2 \
    libcap krb5-libs lz4-libs snappy libmaxminddb gnutls brotli \
    libzstd opus \
    qt5-qtbase qt5-qtmultimedia qt5-qtsvg \
    xdg-utils hicolor-icon-theme desktop-file-utils \
    shadow-utils

# Copy your wireshark RPMs into the same directory
cp /path/to/wireshark-el8-rpms/*.rpm rpms/

# Transfer the rpms/ directory to the offline machine
tar czf wireshark-el8-offline.tar.gz rpms/
```

### 2.3 Installing on the offline machine

```bash
tar xzf wireshark-el8-offline.tar.gz
cd rpms

# Install everything (rpm resolves deps from the local directory)
rpm -Uvh *.rpm

# OR use dnf in offline mode (preferred – handles ordering automatically)
dnf install --disablerepo='*' \
    wireshark-*.rpm \
    wireshark-qt-*.rpm
```

### 2.4 Verifying the installation

```bash
tshark --version
tshark -G protocols | grep -c '^[a-z]'   # prints number of loaded dissectors
tshark -r /path/to/capture.pcap -c 5
```

---

## 3. EL 7 – offline installation (CentOS 7 / RHEL 7)

### 3.1 Custom runtime libraries

The EL7 RPMs are compiled inside a CentOS 7 container where several
under-versioned libraries have been **replaced by newer versions built from
source and installed into `/usr/local`**.  The resulting binaries link against
those newer `.so` versions, which are **not** present in a stock CentOS 7
installation.  You must supply them alongside the RPMs.

| Library | CentOS 7 system version | Version used in build | Installed path |
|---|---|---|---|
| libgpg-error | 1.12 | 1.46 | `/usr/local/lib/libgpg-error.so.0` |
| libgcrypt | 1.5.3 | 1.8.9 | `/usr/local/lib/libgcrypt.so.20` |
| c-ares | 1.10.0 | 1.19.1 | `/usr/local/lib/libcares.so.2` |
| libxml2 | 2.9.1 | 2.9.14 | `/usr/local/lib/libxml2.so.2` |

### 3.2 Creating a deployment bundle (on a connected CentOS 7 machine or in the build container)

Run the EL7 docker build from the workflow once (or reuse an existing CI
artefact), then extract the custom libraries from the container before it
exits.  The simplest method is to add a packaging step:

```bash
# After the build completes, collect the custom libs
tar czf wireshark-el7-custom-libs.tar.gz \
    /usr/local/lib/libgpg-error.so* \
    /usr/local/lib/libgcrypt.so* \
    /usr/local/lib/libcares.so* \
    /usr/local/lib/libxml2.so*
```

Alternatively, build each library from source on the offline target using
the same steps recorded in the workflow (see `.github/workflows/rpm.yml`,
the "Build EL7 RPMs in CentOS 7 container" step).

### 3.3 Installing on the offline CentOS 7 machine

```bash
# 1. Install the custom libraries
tar xzf wireshark-el7-custom-libs.tar.gz -C /
echo /usr/local/lib > /etc/ld.so.conf.d/local.conf
ldconfig

# 2. Install system-level deps (pre-downloaded with yum --downloadonly or repotrack)
rpm -Uvh shadow-utils-*.rpm glib2-*.rpm libpcap-*.rpm zlib-*.rpm \
         pcre2-*.rpm krb5-libs-*.rpm speexdsp-*.rpm \
         lz4-*.rpm snappy-*.rpm gnutls-*.rpm libcap-*.rpm \
         qt5-qtbase-*.rpm qt5-qtmultimedia-*.rpm qt5-qtsvg-*.rpm

# 3. Install Wireshark RPMs
rpm -Uvh wireshark-*.rpm wireshark-qt-*.rpm
```

> **Tip:** Use `rpm -qpR wireshark-*.rpm` on the RPM file to get the exact
> list of `.so` requirements before transferring to the offline machine.

### 3.4 Verifying the installation

```bash
LD_LIBRARY_PATH=/usr/local/lib tshark --version
tshark -G protocols | grep -c '^[a-z]'
```

---

## 4. EL 6 – offline installation (CentOS 6)

### 4.1 Custom runtime libraries

EL6 requires even more custom libraries because CentOS 6 ships significantly
older versions.  The full list of custom libraries installed to `/usr/local`:

| Library | Reason |
|---|---|
| cmake 3.28 | Build tool only – not needed at runtime |
| glib2 2.54.3 | CentOS 6 ships 2.28; Wireshark requires ≥ 2.54 |
| pcre2 10.42 | Not available in CentOS 6 / EPEL 6 at all |
| libxml2 2.9.14 | CentOS 6 ships 2.7.6; Wireshark requires ≥ 2.9.7 |
| libgpg-error 1.46 | Prerequisite for libgcrypt 1.8 |
| libgcrypt 1.8.9 | CentOS 6 ships 1.4.5; Wireshark requires ≥ 1.8.0 |
| c-ares 1.19.1 | CentOS 6 ships 1.7.0; Wireshark requires ≥ 1.13.0 |
| devtoolset-7 GCC 7 runtime libs | Required by binaries compiled with GCC 7 (`/opt/rh/devtoolset-7/root/usr/lib64/`) |

### 4.2 Creating a deployment bundle

Inside the CentOS 6 build container (see workflow), after all libraries are
built and installed:

```bash
tar czf wireshark-el6-runtime-libs.tar.gz \
    /usr/local/lib/libglib-2.0.so* \
    /usr/local/lib/libgobject-2.0.so* \
    /usr/local/lib/libgmodule-2.0.so* \
    /usr/local/lib/libgthread-2.0.so* \
    /usr/local/lib/libpcre2-8.so* \
    /usr/local/lib/libxml2.so* \
    /usr/local/lib/libgpg-error.so* \
    /usr/local/lib/libgcrypt.so* \
    /usr/local/lib/libcares.so* \
    /opt/rh/devtoolset-7/root/usr/lib64/libstdc++.so* \
    /opt/rh/devtoolset-7/root/usr/lib64/libgcc_s.so*
```

### 4.3 Installing on the offline CentOS 6 machine

```bash
# 1. Install the bundled custom libraries
tar xzf wireshark-el6-runtime-libs.tar.gz -C /
echo /usr/local/lib                             > /etc/ld.so.conf.d/local.conf
echo /opt/rh/devtoolset-7/root/usr/lib64       >> /etc/ld.so.conf.d/devtoolset7.conf
ldconfig

# 2. Install available CentOS 6 system deps (pre-downloaded with yum)
rpm -Uvh shadow-utils-*.rpm libpcap-*.rpm zlib-*.rpm \
         libcap-*.rpm krb5-libs-*.rpm gnutls-*.rpm

# 3. Install Wireshark RPMs (Qt GUI not built for EL6)
rpm -Uvh wireshark-*.rpm
```

> **Note:** The `wireshark-qt` package is not produced for EL6 because Qt 5
> is not available in CentOS 6 / EPEL 6 repositories.

---

## 5. Offline execution of the functional test suite

The test suite lives in the `test/` directory of the Wireshark source tree.
All tests in `suite_unittests.py`, `suite_dfilter/`, and `suite_dissectors/`
read **local capture files** from `test/captures/` and run **compiled
binaries** from the build tree.  They require **no network access**.

### 5.1 Required files

You need the following on the offline machine:

1. **Wireshark source tree** (or at minimum the `test/` directory)  
   Transfer from a connected machine or extract from the source tarball
   produced by `cmake --build build --target dist`.

2. **Built binaries** in `<build-dir>/run/`  
   Either transfer the cmake build directory, or re-build from source on
   the offline machine using the same cmake configure command.

3. **Python 3.6+** (Python 3.4 in CentOS 6 is insufficient – see §5.4).

4. **pytest ≥ 3.0** and **pytest-xdist** (optional but recommended for
   parallel execution).

### 5.2 Installing test dependencies offline

**On a connected machine**, download the Python packages as wheels:

```bash
pip3 download --dest ./test-wheels pytest pytest-xdist
tar czf test-wheels.tar.gz test-wheels/
```

**On the offline machine**:

```bash
tar xzf test-wheels.tar.gz
pip3 install --no-index --find-links=./test-wheels pytest pytest-xdist
```

For EL8 with no pip, use RPMs instead:

```bash
# On connected machine
dnf download --resolve --destdir=./pytest-rpms \
    python3-pytest python3-pytest-xdist

# On offline machine
rpm -Uvh ./pytest-rpms/*.rpm
```

### 5.3 Running the test suite

From the **Wireshark source root** (the directory that contains `test/`):

```bash
export WIRESHARK_RUN_FROM_BUILD_DIRECTORY=1

# Run unit + display-filter + dissector suites (no capture, no GUI):
python3 -m pytest \
    --program-path /path/to/build/run \
    --disable-capture \
    --disable-gui \
    --override-ini="addopts=-ra" \
    -v test/suite_unittests.py \
       test/suite_dfilter \
       test/suite_dissectors

# Run the C unit-test programs directly (works on all Python versions):
WIRESHARK_RUN_FROM_BUILD_DIRECTORY=1
for prog in exntest fifo_string_cache_test oids_test reassemble_test \
            tvbtest wmem_test wscbor_test wscbor_enc_test \
            test_epan test_wsutil; do
    /path/to/build/run/$prog && echo "PASS: $prog"
done
```

### 5.4 EL6 limitation – Python 3.4 and f-strings

CentOS 6 ships Python 3.4 via `python34` from EPEL 6.  Python 3.4 cannot
parse **f-strings** (`f"..."`) which are used in `test/conftest.py`.  This
makes the pytest runner itself unable to load the test configuration on EL6.

**Workaround A (recommended):** Install Python 3.6+ from a Software
Collection or a third-party source (e.g., `centos-release-scl` +
`rh-python36`).

```bash
# On CentOS 6 with SCL (requires vault repos configured as per the workflow):
yum install -y rh-python36
source /opt/rh/rh-python36/enable
pip3 install pytest pytest-xdist
python3 -m pytest --program-path /path/to/build/run \
    --disable-capture --disable-gui \
    test/suite_unittests.py test/suite_dfilter test/suite_dissectors
```

**Workaround B:** Run only the compiled C unit-test programs directly
(no pytest required):

```bash
WIRESHARK_RUN_FROM_BUILD_DIRECTORY=1
for prog in exntest fifo_string_cache_test oids_test reassemble_test \
            tvbtest wmem_test wscbor_test wscbor_enc_test \
            test_epan test_wsutil; do
    /path/to/build/run/$prog && echo "PASS: $prog"
done
```

### 5.5 Running tests against installed RPMs (post-install validation)

If you want to run tshark-based tests against the **installed** system binary
rather than the cmake build tree, point `--program-path` at the installation
prefix and disable the run-from-build-tree mode:

```bash
# EL8 – tshark is installed to /usr/sbin by default
python3 -m pytest \
    --program-path /usr/sbin \
    --disable-capture \
    --disable-gui \
    --override-ini="addopts=-ra" \
    test/suite_clopts.py \
    test/suite_dfilter \
    test/suite_dissectors
```

> **Note:** `WIRESHARK_RUN_FROM_BUILD_DIRECTORY` must **not** be set when
> running against an installed binary – it tells tshark to look for plugins
> and data files relative to the binary location rather than the standard
> system paths.

---

## 6. Quick-reference checklist

### Offline install

- [ ] Download RPM artefact bundle from the CI workflow run  
- [ ] For EL7/EL6: bundle custom `/usr/local/lib` libraries from the build
      container and copy to the target machine  
- [ ] Run `ldconfig` after placing custom libraries  
- [ ] Install all dependency RPMs with `rpm -Uvh` or `dnf install --disablerepo='*'`  
- [ ] Verify: `tshark --version`

### Offline test run

- [ ] Transfer the Wireshark source tree (`test/` directory + `test/captures/`)  
- [ ] Transfer the cmake build directory (`build/run/` binaries)  
- [ ] Install pytest offline (wheels or RPMs)  
- [ ] Run: `python3 -m pytest --program-path build/run --disable-capture --disable-gui test/suite_unittests.py test/suite_dfilter test/suite_dissectors`  
- [ ] EL6 only: use `rh-python36` SCL or run C programs directly
