# NS8 testing suite

## Requisites

The are create using [Robot Framework](https://robotframework.org/) and the execution is made inside a Podman container. The only dependences required
in the host system is Podman (https://podman.io/getting-started/installation)

## Tests structure

All test are in the `tests` directory of each NS8 module, in the root of the `tests` directory the file `pythonreq.txt`
specify the dependences required and each subdirectory contains a test suite whit the relative tests.

The test suite and the relative tests are executed in alphabetical order, the `__init__.robot` can be used for the suite
initialization. More details in the [Robot Framework documentation](https://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html#files-and-directories)

```
<module>/tests/
├── 00_<test_suite>
│   ├── 00_<test>.robot
│   ├── 10_<test>.robot
│   ├── 20_<test>.robot
│   ├── ...
│   ├── ...
│   └── __init__.robot
├── 01_<test_suite>
│   ├── 00_<test>.robot
│   ├── 10_<test>.robot
│   ├── 20_<test>.robot
│   ├── ...
│   ├── ...
│   └── __init__.robot
├── ...
│   ├── ...
│   ├── ...
│   └── __init__.robot
├── pythonreq.txt
└── __init__.robot
```
## Tests execution

Every module have a script named `test-module.sh` that can be used to setup and launch the tests relative to the module. All the tests logs can be found in the directory `tests/outputs`

### Usage

All the scripts have a some common way to customize the execution.

    ./test-module.sh <leader_node> <worker_node1,worker_node2,...>

#### Parameters

* `<leader_node>`: The host of the leader node.
* `<worker_node1,worker_node2,...>`: A list of comma separated workers hosts.

#### Environments variables

* `SSH_KEYFILE`: SSH private key to use for connection to the NS8 cluster, default  `~/.ssh/id_rsa`.
* `COREBRANCH`: NS8 branch name
* `COREMODULES`: Comma separated list of modules to install from the selected branch.

The `COREBRANCH` and `COREMODULES` have the same meaning of the [`install.sh`](docs/quickstart.md#install-a-development-branch) parameters.

## Testing environment

A `terraform` configuration for create a clean infrastucture for the tests execution can be found in the [`infra/`](infra/) directory.