---
authors: Blake Devcich <blake.devcich@hpe.com>
categories: provisioning
---

# Data Movement Configuration

Data Movement can be configured in multiple ways:

1. Server side
2. Per Copy Offload API Request arguments

The first method is a "global" configuration - it affects all data movement operations. The second
is done per the Copy Offload API, which allows for some configuration on a per-case basis, but is
limited in scope. Both methods are meant to work in tandem.

## Server Side ConfigMap

The server side configuration is done via the `nnf-dm-config` config map:

```bash
kubectl -n nnf-dm-system get configmap nnf-dm-config
```

The config map allows you to configure the following:

|Setting|Description|
|-------|-----------|
|slots|The number of slots specified in the MPI hostfile. A value less than 1 disables the use of slots in the hostfile.|
|maxSlots|The number of max_slots specified in the MPI hostfile. A value less than 1 disables the use of max_slots in the hostfile.|
|command|The full command to execute data movement. More detail in the following section.|
|progressIntervalSeconds|interval to collect the progress data from the `dcp` command.|

### `command`

The full data movement `command` can be set here. By default, Data Movement uses `mpirun` to run
`dcp` to perform the data movement. Changing the `command` is useful for tweaking `mpirun` or `dcp` options or to
replace the command with something that can aid in debugging (e.g. `hostname`).

`mpirun` uses hostfiles to list the hosts to launch `dcp` on. This hostfile is created for each Data
Movement operation, and it uses the config map to set the `slots` and `maxSlots` for each host (i.e. NNF
node) in the hostfile. The number of `slots`/`maxSlots` is the same for every host in the hostfile.

Additionally, Data Movement uses substitution to fill in dynamic information for each Data Movement
operation. Each of these **must** be present in the command for Data Movement to work properly when
using `mpirun` and `dcp`:

|VAR|Description|
|---|-----------|
|`$HOSTFILE`|hostfile that is created and used for mpirun.|
|`$UID`|User ID that is inherited from the Workflow.|
|`$GID`|Group ID that is inherited from the Workflow.|
|`$SRC`|source for the data movement.|
|`$DEST`|destination for the data movement.|

By default, the command will look something like the following. Please see the config map itself for
the most up to date default command:

```bash
mpirun --allow-run-as-root --hostfile $HOSTFILE dcp --progress 1 --uid $UID --gid $GID $SRC $DEST
```

### Profiles

**Note:** This feature is not fully implemented, but is present in the `nnf-dm-config` config map.
Only the `default` profile is used, and it must be present in the configuration. Once the feature is
implemented, a user will be able to select a profile using #DW directives and/or the Copy Offload
API. Right now, users can add additional profiles into the config map, but the only way to use them
would be to rename one of them to be `default`.

`slots`, `maxSlots`, and `command` can be stored in Data Movement profiles. These profiles are
available for a quick way to switch between different settings for a particular workflow.

Example profile:

```yaml
profiles:
  default:
      slots: 8
      maxSlots: 0
      command: mpirun --allow-run-as-root --hostfile $HOSTFILE dcp --progress 1 --uid $UID --gid $GID $SRC $DEST
```

## Copy Offload API Daemon

The `CreateRequest` API call that is used to create Data Movement with the Copy Offload API has some
options to allow a user to specify some options for that particular Data Movement. These settings
are on a per-request basis.

See the [CreateRequest API](https://github.com/NearNodeFlash/nnf-dm/blob/master/daemons/compute/Readme.md#create-request)
definition for what can be configured.
