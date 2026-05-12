# Intermediate & Advanced `uv` Notes

## Table of Contents

1. [Import `uv` into a Docker Image](#1-import-uv-into-a-docker-image)
2. [Install Optional Dependencies from a `pyproject.toml`](#2-install-optional-dependencies-from-a-pyprojecttoml)
3. [Install Dependency Groups](#3-install-dependency-groups)
4. [Install a Library from a Specific Source](#4-install-a-library-from-a-specific-source)
5. [PyTorch — Install from a Specific Source and Index](#5-pytorch--install-from-a-specific-source-and-index)
6. [Prevent the Source from Being Installed as a Package](#6-prevent-the-source-from-being-installed-as-a-package)

---

## 1. Import `uv` into a Docker Image

To include `uv` in a Docker image, use a multi-stage `COPY` instruction to pull the binary directly from the official `uv` image:

```dockerfile
COPY --from=ghcr.io/astral-sh/uv:0.11.13 /uv /usr/local/bin
```

This copies the `uv` binary into `/usr/local/bin`, making it available system-wide without the need for a separate installation step.

---

## 2. Install Optional Dependencies from a `pyproject.toml`

Optional dependency groups are declared under `[project.optional-dependencies]` in `pyproject.toml`. This is useful for separating development tools, testing frameworks, or other extras from the core runtime dependencies.

**Example `pyproject.toml`:**

```toml
[project]
name = "ProjectName"
requires-python = ">=3.11"
dependencies = [
    "numpy==1.26.4",
    "pandas>=2.1.0",
    "openpyxl>=3.1.3",
    "matplotlib",
    "seaborn",
]

[project.optional-dependencies]
dev = [
    "ipykernel>=6.30.0",
    "ruff>=0.13.0",
    "pytest>=8.3.0",
    "pytest-cov>=6.0.0",
    "pre-commit>=4.1.0",
]
```

To install a **specific** optional group, use the `--extra` flag:

```bash
sudo uv pip install --system --extra=dev -r pyproject.toml
```

To install **all** optional groups at once, use `--all-extras`:

```bash
sudo uv pip install --system --all-extras -r pyproject.toml
```

---

## 3. Install Dependency Groups

`uv` also supports `[dependency-groups]`, a more flexible way to organize sets of dependencies that are not strictly tied to the package itself — for example, different model stacks or experimentation environments.

**Example `pyproject.toml`:**

```toml
[project]
name = "ProjectName"
dependencies = [
    "pandas",
    "pyarrow",
    "matplotlib>=3.10.7",
    "seaborn>=0.13.2",
]

[dependency-groups]
rn = [
    "scikit-learn>=1.6.0",
    "torch==2.8.0",
    "lightgbm>=4.0.0",
    "torchinfo>=1.8.0",
]

me = [
    "scikit-learn",
    "optuna",
    "deap",
    "optunahub",
]
```

To install a specific group, use the `--group` flag:

```bash
sudo uv pip install --group=rn
```

> **Note:** Unlike `[project.optional-dependencies]`, dependency groups are a `uv`-specific concept and are not part of the PEP 508 standard.

---

## 4. Install a Library from a Specific Source

`uv` allows you to pin a dependency to a custom source — such as a private Git repository — using the `[tool.uv.sources]` table. This is particularly useful for internal libraries hosted on platforms like Azure DevOps.

**Example `pyproject.toml`:**

```toml
[project]
name = "ProjectName"
requires-python = ">=3.11"
dependencies = [
    "telefonipy",
    "numpy==1.26.4",
    "pandas>=2.1.0",
    "openpyxl>=3.1.3",
    "mlflow==2.22.2",
    "boto3>=1.40.0",
]

[tool.uv.sources]
telefonipy = { git = "https://CelulaAnaliticaAvanzada@dev.azure.com/CelulaAnaliticaAvanzada/libraries/_git/telefonipy", rev = "v0.0.1" }
```

The `rev` field pins the dependency to a specific tag, branch, or commit hash, ensuring reproducible installs.

---

## 5. PyTorch — Install from a Specific Source and Index

Some packages, like PyTorch, are distributed through custom package indexes rather than PyPI. `uv` supports this via `[tool.uv.sources]` combined with `[[tool.uv.index]]` entries.

**Example `pyproject.toml`:**

```toml
[project]
name = "unsloth-env"
version = "0.1.0"
dependencies = [
    "unsloth==2026.1.4",
    "torch==2.9.0",
    "torchvision==0.24.0",
    "torchaudio",

    "transformers==4.56.2",
    "tokenizers==0.22.2",
    "sentencepiece==0.2.1",
    "protobuf==5.29.5",
    "trl==0.22.2",
    "peft==0.18.1",
    "accelerate==1.12.0",
    "bitsandbytes==0.49.1",
    "triton==3.5.0",
    "safetensors==0.7.0",
    "xformers==0.0.33.post1",

    "scikit-learn>=1.8.0",
    "azure-storage-blob==12.28.0",
]

[project.optional-dependencies]
dev = [
    "pip",
    "ipykernel",
    "jupyter",
]

[tool.uv.sources]
torch = { index = "pytorch-cu126" }
torchvision = { index = "pytorch-cu126" }
torchaudio = { index = "pytorch-cu126" }
xformers = { index = "pytorch-cu126" }

[[tool.uv.index]]
name = "pytorch-cu126"
url = "https://download.pytorch.org/whl/cu126"
explicit = true
```

Setting `explicit = true` ensures that the custom index is only used for the packages that explicitly reference it, preventing unintended resolution from that index for other dependencies.

---

## 6. Prevent the Source from Being Installed as a Package

[📖 Astral Docs](https://docs.astral.sh/uv/reference/settings/#package)

By default, `uv` inspects the presence of a `[build-system]` declaration to decide whether the project itself should be built and installed into the environment. This behavior can be overridden using the `tool.uv.package` setting:

- **`package = true`** — Forces the project to be built and installed, even if no build system is explicitly configured. If no build system is defined, `uv` will fall back to the `setuptools` legacy backend.
- **`package = false`** — Prevents the project from being built and installed into the environment. `uv` will ignore the declared build system during normal operations, though explicit build commands like `uv build` are still respected.

This is especially handy in ML or data science projects where the repository is not meant to be distributed as a Python package, but a `[build-system]` block is present for other tooling reasons.

**Example `pyproject.toml`:**

```toml
[build-system]
requires = ["setuptools"]
build-backend = "setuptools.build_meta"

# ...

[tool.uv]
managed = true
package = false
override-dependencies = [
    "torch==2.9.0",
    "transformers==4.56.2",
    "trl==0.22.2",
    "xformers==0.0.33.post1",
]
```

The `override-dependencies` field can also be used here to force specific versions of transitive dependencies, overriding whatever versions the resolver would otherwise choose.