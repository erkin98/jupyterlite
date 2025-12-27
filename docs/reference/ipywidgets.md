# IPywidgets Support

This section describes how [ipywidgets](https://ipywidgets.readthedocs.io/) support is
implemented in JupyterLite and how interactive widgets work in the browser.

## Overview

Interactive widgets in Jupyter enable bidirectional communication between the frontend
(JavaScript in the browser) and the kernel (Python code). In JupyterLite, this
communication happens entirely in the browser, with the kernel running in a Web Worker.

The key components that enable ipywidgets support in JupyterLite are:

1. **The Comm Protocol**: A message-passing protocol that enables communication between
   the kernel and the frontend.
2. **The JupyterLab Widget Manager**: The `@jupyter-widgets/jupyterlab-manager`
   federated extension that renders widgets in the frontend.
3. **The ipywidgets Python Package**: The Python library installed in the kernel that
   provides widget functionality.

## Architecture

```{mermaid}
flowchart LR
    subgraph Browser
        subgraph main thread
            direction TB
            M[JupyterLab UI]
            M --- WM[Widget Manager]
            WM --- V[Widget Views]
        end
        subgraph web worker
            direction TB
            K[Kernel]
            K --- IW[ipywidgets]
            IW --- C[Comm]
        end
    end
    C <--->|Comm Messages| WM
```

### Communication Flow

When a user creates a widget in Python code (e.g., `ipywidgets.IntSlider()`), the
following communication flow occurs:

```{mermaid}
sequenceDiagram
    participant P as Python Code
    participant K as Kernel
    participant M as Main Thread
    participant W as Widget Manager

    P->>+K: Create widget (IntSlider())
    K->>+M: comm_open message
    M->>+W: Create model
    W->>W: Create view
    W-->>-M: Widget rendered
    M-->>-K: comm_msg (ack)
    K-->>-P: Widget displayed
```

### The Comm Protocol

The Comm (communication) protocol is the foundation for widget communication. It enables
bidirectional, asynchronous communication between the kernel and the frontend.

JupyterLite implements the Comm protocol in `@jupyterlite/services`. The
[`BaseKernel`](https://jupyterlite.readthedocs.io/en/stable/reference/api/ts/classes/jupyterlite_services.BaseKernel.html)
class handles three types of comm messages:

- **`comm_open`**: Opens a new communication channel for a widget
- **`comm_msg`**: Sends data through an existing channel (e.g., widget state updates)
- **`comm_close`**: Closes a communication channel

When a widget's state changes (e.g., a slider is moved), a `comm_msg` is sent to
synchronize the state between the frontend and the kernel.

### Widget State Synchronization

Widgets maintain a synchronized state between the Python kernel and the JavaScript
frontend:

```{mermaid}
sequenceDiagram
    participant V as Widget View (JS)
    participant M as Widget Model (JS)
    participant K as Kernel (Python)
    participant W as Widget (Python)

    Note over V,W: User moves slider in UI
    V->>M: Update model value
    M->>K: comm_msg (state change)
    K->>W: Update Python widget value
    Note over V,W: Python code changes value
    W->>K: Send state update
    K->>M: comm_msg (state change)
    M->>V: Update view
```

## Setting up IPywidgets

### With the Pyodide Kernel

When using the [jupyterlite-pyodide-kernel](https://github.com/jupyterlite/pyodide-kernel),
ipywidgets needs to be:

1. **Installed as a frontend extension**: Add `ipywidgets` to your build environment's
   `requirements.txt` to include the `@jupyter-widgets/jupyterlab-manager` federated
   extension.

2. **Installed at runtime in the kernel**: Users install ipywidgets in their notebooks:

   ```python
   %pip install -q ipywidgets
   ```

   or using piplite directly:

   ```python
   import piplite
   await piplite.install("ipywidgets")
   ```

### With the Xeus Python Kernel

When using [jupyterlite-xeus](https://github.com/jupyterlite/xeus) with xeus-python,
ipywidgets can be pre-installed by specifying it in the `environment.yml` file:

```yaml
name: xeus-python-kernel
channels:
  - https://repo.mamba.pm/emscripten-forge
  - conda-forge
dependencies:
  - xeus-python
  - ipywidgets
```

This automatically includes both the Python package and the frontend extension.

## How Third-Party Widget Libraries Work

Libraries like [bqplot](https://github.com/bqplot/bqplot),
[ipyleaflet](https://github.com/jupyter-widgets/ipyleaflet), and
[plotly](https://github.com/plotly/plotly.py) extend ipywidgets to provide
specialized visualizations.

These libraries require:

1. **A federated JupyterLab extension**: For rendering the custom widgets in the browser.
2. **A Python package**: For the kernel-side widget implementation.

See [the extensions configuration guide](../howto/configure/simple_extensions.md) for details on adding widget
libraries to your JupyterLite deployment.

## Limitations

While ipywidgets support works well in JupyterLite, there are some considerations:

- **Web Worker Environment**: The kernel runs in a Web Worker, which has limitations
  compared to a full Python environment. Some advanced widget features may not work.

- **Version Compatibility**: The frontend extension version must be compatible with the
  Python package version installed at runtime. See
  [version management strategies](../howto/configure/simple_extensions.md#avoid-the-drift-of-versions-between-the-frontend-extension-and-the-python-package)
  for guidance on managing version compatibility.

- **Package Availability**: Not all Python packages are available for Pyodide or
  emscripten-forge. Check the respective package indexes for availability.

## Further Reading

- [IPywidgets Documentation](https://ipywidgets.readthedocs.io/)
- [Adding Extensions](../howto/configure/simple_extensions.md)
- [Pyodide Kernel Configuration](../howto/pyodide/packages.md)
- [Xeus Python Pre-installed Packages](../howto/xeus-python/preinstalled_packages.md)
- [Jupyter Widget Ecosystem](https://jupyter.org/widgets)
