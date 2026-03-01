# Rubin LSST Stack + RC2 Setup (Concise Tutorial)

This is a minimal, reproducible setup that worked on macOS with `mamba`, `git-lfs`, and a pre-existing conda installation.

## 1. Prerequisites

Install or verify:

- `mamba` and `conda`
- `git` and `git-lfs`

```bash
mamba --version
conda --version
git --version
git lfs version
```

If missing, install them:

- If both `conda` and `mamba` are missing, install Miniforge first (provides `conda`):
  ```bash
  curl -L -o Miniforge3.sh \
    https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh
  bash Miniforge3.sh -b -p "$HOME/miniforge3"
  source "$HOME/miniforge3/etc/profile.d/conda.sh"
  conda init bash
  exec bash
  ```
- Then install `mamba`:
  ```bash
  conda install -n base -c conda-forge mamba
  ```
- `git-lfs` (recommended via Homebrew):
  ```bash
  brew install git-lfs
  git lfs install
  ```

Create the `stack/` directory in your current directory, then bootstrap Rubin:

```bash
mkdir -p stack
curl -fL https://ls.st/lsstinstall -o stack/lsstinstall
chmod +x stack/lsstinstall
(cd stack && ./lsstinstall)
```

This creates `stack/loadLSST.sh` and configures the Rubin conda env (for example `lsst-scipipe-12.1.0`).

## 2. Activate and install `lsst_distrib`

```bash
source stack/loadLSST.sh
eups distrib install -t w_latest lsst_distrib
```

Notes:

- For classes, you may prefer a fixed tag (for example `-t w_2026_07` or a release tag) instead of `w_latest`.
- This step can take a while.

## 3. Install `rc2_subset` into `rc2/`

```bash
GIT_LFS_SKIP_SMUDGE=1 git clone --depth 1 https://github.com/lsst/rc2_subset.git rc2
cd rc2
git lfs pull
cd ..
```

Expected Butler repo root: `rc2/SMALL_HSC`.

## 4. Install JupyterLab and register a kernel (clean-machine safe)

```bash
source stack/loadLSST.sh
mamba install -n "$LSST_CONDA_ENV_NAME" -y jupyterlab ipykernel
python -m ipykernel install --user --name lsst-pipelines --display-name "LSST (pipelines)"
```

## 5. Configure a Jupyter kernel that always sets up `lsst_distrib`

Create wrapper script:

```bash
cat > stack/python-lsst-pipelines.sh <<'EOF'
#!/usr/bin/env bash
set -eo pipefail
source "$(cd "$(dirname "$0")" && pwd)/loadLSST.sh"
setup lsst_distrib
exec python "$@"
EOF
chmod +x stack/python-lsst-pipelines.sh
```

Find kernel location programmatically from `jupyter kernelspec list --json`, then patch only the interpreter line in `kernel.json`:

```bash
source stack/loadLSST.sh
jupyter kernelspec list --json

KERNEL_DIR="$(python - <<'PY'
import json, subprocess
data = json.loads(subprocess.check_output(["jupyter", "kernelspec", "list", "--json"], text=True))
print(data["kernelspecs"]["lsst-pipelines"]["resource_dir"])
PY
)"
echo "Using kernelspec dir: $KERNEL_DIR"

sed -i.bak \
  "s|\"argv\": \\[[[:space:]]*\"[^\"]*\"|\"argv\": [\"$(pwd)/stack/python-lsst-pipelines.sh\"|" \
  "$KERNEL_DIR/kernel.json"
```

## 6. Verify setup

```bash
source stack/loadLSST.sh
setup lsst_distrib
python - <<'PY'
from lsst.daf.butler import Butler
b = Butler("rc2/SMALL_HSC")
print("collections:", len(list(b.registry.queryCollections())))
print("has defaults:", "HSC/RC2/defaults" in list(b.registry.queryCollections()))
ref = next(iter(b.registry.queryDatasets("raw", collections="HSC/RC2/defaults").expanded()))
print("example raw dataId:", dict(ref.dataId.mapping))
PY
```

## 7. Start JupyterLab

```bash
source stack/loadLSST.sh
jupyter lab
```

In notebooks, select kernel: **`LSST (pipelines)`**.

## 8. Manual end-to-end notebook test

Use JupyterLab interactively:

1. Start JupyterLab:
   ```bash
   source stack/loadLSST.sh
   jupyter lab
   ```
2. Open `demo_rc2_jupyter.ipynb`.
3. Set kernel to **`LSST (pipelines)`**.
4. Run all cells (`Run -> Run All Cells`).

Expected result:

- the notebook prints `Demo OK`
- one raw RC2 detector image is displayed

## 9. Common issues

- `git lfs pull` appears slow: large data transfer is normal for RC2.
- Kernel wrapper fails with `GFORTRAN ... unbound variable`: do **not** use `set -u` in the wrapper script.
- If a kernel points to an old stack path, re-run Step 5 to re-discover `resource_dir` from JSON and rewrite that `kernel.json`.
