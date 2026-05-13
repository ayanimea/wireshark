# Wireshark RPM – Offline / Standalone Installation and Testing

This document explains how to install and test Wireshark on a machine that is
**completely offline** and has **no compiler** — only `rpm` and `bash` are
required for installation and smoke tests (tests 1–4).  The full pytest suite
(test 5) additionally requires **python3 ≥ 3.7** and **pip**
(`python3 -m pip`).

---

## 1. What the CI produces

Each successful `Build EL RPMs` workflow run uploads three artifact bundles:

| Artifact | OS | Contents |
|---|---|---|
| `wireshark-el8-rpms` | AlmaLinux 8 | RPMs + `install.sh` + `test-install.sh` + `pytest.ini` + `wheels/` + `test/` |
| `wireshark-el7-rpms` | CentOS 7 (devtoolset-9) | RPMs + `install.sh` + `test-install.sh` + `pytest.ini` + `wheels/` + `test/` + `custom-libs.tar.gz` |
| `wireshark-el6-rpms` | CentOS 6 (devtoolset-7) | RPMs + `install.sh` + `test-install.sh` + `pytest.ini` + `test/` + `custom-libs.tar.gz` (no `wheels/`) |

`custom-libs.tar.gz` (EL7/EL6 only) contains the newer runtime libraries that
the distro ships in too-old a version — `libgcrypt`, `libgpg-error`, `c-ares`,
`libxml2`, `libstdc++`, and (EL6 only) `glib2`, `pcre2`.  Extracting it is
handled automatically by `install.sh`.

`wheels/` contains a pre-downloaded `pytest==6.2.5` wheel archive so the test
suite can be installed with `pip install --no-index` — no PyPI access needed.

`test/` contains the Wireshark test suite (display-filter tests, dissector
tests, CLI-option tests) and all required capture files.

---

## 2. Installation (all EL variants — two commands)

### Step 1 — transfer the bundle

Download the CI artifact `.zip` on any internet-connected machine, then copy
the unzipped directory to the target via USB, SCP, or similar:

```bash
unzip wireshark-el8-rpms.zip   # or el7 / el6
```

### Step 2 — install (run as root on the target)

```bash
sudo bash install.sh
```

What `install.sh` does, in order:

| EL8 | EL7 / EL6 |
|---|---|
| `rpm -Uvh wireshark*.rpm` | Extract `custom-libs.tar.gz` → `/usr/local/lib/` |
| | `ldconfig` (registers the bundled libs) |
| | `rpm -Uvh wireshark*.rpm` |

`rpm -Uvh` resolves cross-package dependencies within the bundle (e.g.
`wireshark-qt` depends on `wireshark`).  All shared libraries that `tshark`
needs at runtime are either part of the standard base OS install or are
provided by `custom-libs.tar.gz`.

> **EL8 note** — if the target has never had EPEL configured, `rpm -Uvh` may
> report a missing dependency such as `libspeexdsp.so.1`.  In that case either
> copy the matching `speexdsp` RPM from an EL8 mirror into the bundle directory
> before running `install.sh`, or skip the `wireshark-qt` package and install
> only the base `wireshark` RPM:
>
> ```bash
> sudo rpm -Uvh wireshark-[0-9]*.rpm   # tshark + plugins, no Qt GUI
> ```

---

## 3. Testing (no internet, no compiler required)

```bash
bash test-install.sh
```

The script runs five checks against the **installed** `/usr/bin/tshark` binary:

| # | Test | What it verifies |
|---|---|---|
| 1 | `tshark --version` | Binary runs and links correctly |
| 2 | `tshark -G protocols` | Dissector database loads (> 100 protocols) |
| 3 | ARP dissection | Inline ARP pcap decoded; `arp.src.hw_mac` field correct |
| 4 | Display filter | `arp.opcode == 1` filter matches the inline frame |
| 5 | pytest suite | `suite_dfilter`, `suite_dissectors`, `suite_clopts` run against installed binaries |

Tests 1–4 require only `bash` and `tshark`.

Test 5 requires **python3 ≥ 3.7** and **pip**.  It installs `pytest` from the
bundled `wheels/` directory — completely offline:

```
python3 -m pip install --no-index --find-links ./wheels pytest
```

On EL6 the standard Python 3 from SCL (`rh-python36`) is version 3.6, which
predates `subprocess.run(capture_output=...)` added in Python 3.7; the
`test-install.sh` detects this and skips test 5 automatically (tests 1–4 still
run).  Additionally, no `wheels/` directory is bundled in the EL6 artifact; to
run test 5 on EL6 you must provide both a Python 3.7+ interpreter and a
`wheels/` directory alongside `test-install.sh`.

### Expected output

> **Note:** The sample below shows the EL7/EL8 case where all 5 tests run.
> On EL6, test 5 is automatically skipped (no `wheels/` bundled; Python 3.7+
> not installed by default), so the summary will show `4 passed, 0 failed`.

```
=== Wireshark post-install tests ===
tshark: /usr/bin/tshark

--- 1. tshark --version ---
TShark (Wireshark) 4.x.y ...
  PASS: tshark --version

--- 2. Protocol database ---
  PASS: 3140 protocols loaded

--- 3. ARP packet dissection ---
  PASS: ARP dissector (MAC: de:ad:be:ef:00:01)

--- 4. Display filter ---
  PASS: display filter (arp.opcode == 1)

--- 5. pytest suite (Python 3.x) ---
  ...
  PASS: pytest suite

=== Results: 5 passed, 0 failed ===
```

---

## 4. Bundle layout reference

```
wireshark-el7-rpms/          (or el8 / el6)
├── install.sh               ← run as root; installs RPMs
├── test-install.sh          ← run as any user; runs tests
├── wireshark-4.x.y-N.el7.x86_64.rpm
├── wireshark-qt-4.x.y-N.el7.x86_64.rpm
├── wireshark-devel-4.x.y-N.el7.x86_64.rpm
├── custom-libs.tar.gz       ← EL7/EL6 only; extracted by install.sh
├── pytest.ini               ← pytest root config (used by test-install.sh)
├── wheels/                  ← EL7/EL8 only; not bundled for EL6
│   ├── pytest-6.2.5-py3-none-any.whl
│   └── <pytest dependencies>-py3-none-any.whl
└── test/
    ├── conftest.py
    ├── matchers.py
    ├── subprocesstest.py
    ├── suite_clopts.py
    ├── suite_dfilter/
    ├── suite_dissectors/
    ├── captures/
    ├── config/
    └── keys/
```

---

## 5. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `error while loading shared libraries: libgcrypt.so.20` | Custom libs not deployed | Re-run `sudo bash install.sh` from the bundle directory |
| `Failed dependencies: libspeexdsp.so.1` (EL8) | `speexdsp` not installed | Copy `speexdsp` RPM into bundle dir and re-run, or install base package only (see §2) |
| pytest step skipped | Python < 3.7 or pip absent or no `wheels/` dir | Tests 1–4 still validate core functionality |
| `tshark: command not found` | RPM not installed | Run `sudo bash install.sh` first |
