# Hooks

Heptio Ark currently supports executing commands in containers in pods during a backup.

## Backup Hooks

When performing a backup, you can specify one or more commands to execute in a container in a pod
when that pod is being backed up.

Ark versions prior to v0.7.0 only support hooks that execute prior to any custom action processing
("pre" hooks).

As of version v0.7.0, Ark also supports "post" hooks - these execute after all custom actions have
completed, as well as after all the additional items specified by custom actions have been backed
up.

There are two ways to specify hooks: annotations on the pod itself, and in the Backup spec.

### Specifying Hooks As Pod Annotations

You can use the following annotations on a pod to make Ark execute a hook when backing up the pod:

#### Pre hooks

| Annotation Name | Description |
| --- | --- |
| `pre.hook.backup.ark.heptio.com/container` | The container where the command should be executed.  Defaults to the first container in the pod. Optional. |
| `pre.hook.backup.ark.heptio.com/command` | The command to execute. If you need multiple arguments, specify the command as a JSON array, such as `["/usr/bin/uname", "-a"]` |
| `pre.hook.backup.ark.heptio.com/on-error` | What to do if the command returns a non-zero exit code.  Defaults to Fail. Valid values are Fail and Continue. Optional. |
| `pre.hook.backup.ark.heptio.com/timeout` | How long to wait for the command to execute. The hook is considered in error if the command exceeds the timeout. Defaults to 30s. Optional. |

Ark v0.7.0+ continues to support the original (deprecated) way to specify pre hooks - without the
`pre.` prefix in the annotation names (e.g. `hook.backup.ark.heptio.com/container`).

#### Post hooks (v0.7.0+)

| Annotation Name | Description |
| --- | --- |
| `post.hook.backup.ark.heptio.com/container` | The container where the command should be executed.  Defaults to the first container in the pod. Optional. |
| `post.hook.backup.ark.heptio.com/command` | The command to execute. If you need multiple arguments, specify the command as a JSON array, such as `["/usr/bin/uname", "-a"]` |
| `post.hook.backup.ark.heptio.com/on-error` | What to do if the command returns a non-zero exit code.  Defaults to Fail. Valid values are Fail and Continue. Optional. |
| `post.hook.backup.ark.heptio.com/timeout` | How long to wait for the command to execute. The hook is considered in error if the command exceeds the timeout. Defaults to 30s. Optional. |

### Specifying Hooks in the Backup Spec

Please see the documentation on the [Backup API Type][1] for how to specify hooks in the Backup
spec.

## Hook Example with fsfreeze

We are going to walk through using both pre and post hooks for freezing a file system. Freezing the
file system is useful to ensure that all pending disk I/O operations have completed prior to taking a snapshot.

We will be using [example/nginx-app/with-pv.yaml][2] for this example. Follow the [steps for your provider][3] to
setup this example.

### Annotations

The Ark [example/nginx-app/with-pv.yaml][2] serves as an example of adding the pre and post hook annotations directly
to your declarative deployment. Below is an example of what updating an object in place might look like.

```shell
kubectl annotate pod -n nginx-example -l app=nginx \
    pre.hook.backup.ark.heptio.com/command='["/sbin/fsfreeze", "--freeze", "/var/log/nginx"]' \
    pre.hook.backup.ark.heptio.com/container=fsfreeze \
    post.hook.backup.ark.heptio.com/command='["/sbin/fsfreeze", "--unfreeze", "/var/log/nginx"]' \
    post.hook.backup.ark.heptio.com/container=fsfreeze
```

Now test the pre and post hooks by creating a backup. You can use the Ark logs to verify that the pre and post
hooks are running and exiting without error.

```shell
ark backup create nginx-hook-test

ark backup get nginx-hook-test
ark backup logs nginx-hook-test | grep hookCommand
```


[1]: api-types/backup.md
[2]: examples/nginx-app/with-pv.yaml
[3]: cloud-common.md
