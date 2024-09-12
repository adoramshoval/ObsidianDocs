The make command **compiles different program pieces and builds a final executable**. The purpose of `make` is to automate file compilation, making the process simpler and less time-consuming. The command works with any programming language as long as the compiler can be executed with a shell command.

### How to Compile specific Kubernetes repository components?

In the Kubernetes build system, the `WHAT` variable in the `make` command specifies **which component or binary to build**. It tells the `make` command exactly which files or targets need to be compiled.

#### How `WHAT` Works:

- **`WHAT=<path>`**: The path you specify with `WHAT` points to the target (binary or component) you want to build within the Kubernetes repository.

For example:
- In your case, `make WHAT=test/e2e/e2e.test` tells `make` to compile only the `e2e.test` binary located at `test/e2e/`.
- Without `WHAT`, the `make` command would try to build the entire Kubernetes project, which includes all the binaries such as `kube-apiserver`, `kubelet`, `kubectl`, etc.

#### Example Breakdown:

- `test/e2e/e2e.test`: This refers to the **E2E test binary**. By specifying `WHAT=test/e2e/e2e.test`, you are instructing `make` to compile the Go code from the `test/e2e/` directory into the `e2e.test` binary.

The `WHAT` variable narrows the focus to specific targets, speeding up the build process by avoiding unnecessary compilation of other Kubernetes components.

#### **`WHAT` is Specific to Kubernetes Repository**:
- **Makefiles**: These are automation scripts used by the `make` tool to define how to compile and link programs. Makefiles can define custom variables like `WHAT` for flexible build processes.
- **In Kubernetes**: The `WHAT` variable in Kubernetes' Makefile is a way to specify what binary or component to build. The Kubernetes Makefile is designed to manage building multiple components (e.g., `kube-apiserver`, `kubectl`, `e2e.test`) and `WHAT` helps the build system know which component to focus on.
  
  This is not a general `make` feature, but a Kubernetes-specific convention. You wouldn't find `WHAT` used in a standard `Makefile` unless it was specifically defined by the developers of that project.

#### **Why You Can't Just Use `go build`**:
While `go build` is the standard command to compile Go programs, Kubernetes uses a more complex build system. Here's why:

- **Multiple Components**: Kubernetes consists of many components like `kube-apiserver`, `kubelet`, `kubectl`, and many others. The `Makefile` handles building, packaging, and even cross-compilation for multiple platforms. Simply running `go build` might not correctly handle dependencies, cross-compilation, or build flags that are required for the Kubernetes components.
  
- **Custom Build Rules**: Kubernetes uses a **custom build system** to set specific build flags, version information, and manage dependencies across different components. The `make` system wraps `go build` and automates tasks like setting version strings, managing output locations, and dealing with platform-specific nuances.
  
- **Tooling Integration**: The `make` system in Kubernetes integrates with other tools like `bazel`, `docker`, and `containerd`. Simply using `go build` would miss out on this orchestration.

For example, the `e2e.test` binary might require certain dependencies or environment setups that are handled by the `Makefile` but not by running `go build` alone.

#### **How `make` Knows How to Compile Go Code**:
`make` itself does not inherently know how to compile Go code. It relies on **instructions defined in the `Makefile`** to execute the appropriate commands. The Kubernetes Makefile has specific rules and targets for building Go programs.

Here's how it works in Kubernetes:

- The `Makefile` includes specific commands for building Go binaries using the `go build` tool under the hood. For example, if you check the Kubernetes `Makefile`, you'll see commands like this:
  
  ```make
  build: $(WHAT)
      go build -o $(OUTPUT_BIN) ./$(WHAT)
  ```

  This tells `make` to run `go build` with the specified output directory (`OUTPUT_BIN`) and the component defined by `WHAT`.

In this case:
- `$(WHAT)` refers to the specific binary you want to build (like `test/e2e/e2e.test`).
- `go build -o $(OUTPUT_BIN)` tells `make` to invoke `go build` and place the output binary in the specified directory.