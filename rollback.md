# The rollback feature

Rollbacking is a important feature of software-updates solutions. Automatic rollbacking is
even better. This allows for error handling to be under-the-hood without having to worry
about manually resetting your target to its previous state. In addition, the target can
continue its tasks if it has properly rollbacked.   

## Changing the client feedback behavior

The FullMetalUpdate client was regularly polling the server and downloading updates when
there was one. After completing the download, the client “committed” the update to the
corresponding OSTree repo (containers or OS) and then sent the feedback right afterwards.
The problem with this solution is that we don’t know if the new update has properly
started and if it is properly running without problems on the target.

This was an issue that needed to be fixed, especially since we want to send an error
message if the client has rollbacked onto the previous version because of a failure in the
new one. Let’s detail the behavior of the rollback for OS and container upgrades  while
considering this change of functionality in the client.

## Rollbacking the OS in case of boot failure

To rollback the OS to its previous state  we make use of an OSTree’s core feature. OSTree
always keeps the last two deployments in storage. This allows for storage efficiency and
rollbacking. In fact, if you deployed an update to your target you will explicitly  see
`(rollback)` mentioned next to the previous commit (i.e. the previous version of your OS).
Why not simply make use of that feature you may ask ? Well it turns out OSTree has not
implemented this feature yet (but [plans to][ostree_comment_on_rollback]). Hence, we had
to think of a workaround to make use of that feature.

### How OSTree and U-boot work together

OSTree, at its core, makes use of the kernel command line to know on which deployment it
has to boot.

At boot time, U-boot will load OSTree’s environment variable in `/boot/loader/uEnv.txt`.
This file contains crucial information needed by OSTree to know which file system it will
`chroot()` into : 

```
kernel_image=/ostree/poky-<hash>/vmlinuz
ramdisk_image=/ostree/poky-<hash>/initramfs
bootargs=ostree=/ostree/boot.0/poky/<hash>/0
```

These will be added to the kernel command line and parsed by OSTree.

When a new deployment is added, OSTree will change `/boot/loader/uEnv.txt` to : 

```
kernel_image=/ostree/poky-<hash>/vmlinuz
ramdisk_image=/ostree/poky-<hash>/initramfs
bootargs=ostree=/ostree/boot.1/poky/<hash>/0
kernel_image2=/ostree/poky-<hash>/vmlinuz
ramdisk_image2=/ostree/poky-<hash>/initramfs
bootargs2=ostree=/ostree/boot.1/poky/<hash>/1
```

All the `*2` argument are the previous kernel command line arguments which were used to boot
into the previous deployment. So by playing with *U-boot environment variables* we can make
use of this to rollback to the previous deployment.

### Making use of U-boot environment variables

Before loading the kernel, U-boot loads an environment stored in non-volatile memory. This
is used for specific boot configurations, such as downloading the kernel over TFTP for
example.

The rollback feature consists of trying to boot the new deployment multiple times and
rollbacking in case none of the previous boots have succeeded. Here is the pseudo-code
describing how rollbacking works : 

```
The system boots…
U-boot environment variables are loaded…

if init_var is not set:
  success = 0  # this deployment is not yet successful
  trials = 0   # trials is the number of times we tried to boot
  init_var = 1 # init_var is now set
  save u-boot environment

load the ostree variables

if success = 1:
  boot
elif trials < 5:
  trials = trials + 1
  save u-boot environment
  boot
else
  replace bootargs by bootargs2
  delete init_var
  save u-boot environment
  boot
end if
```

### How OS rollbacking works (all the steps)

#### Fetching the new deployment and resetting U-boot’s environment

As expected, the first thing we do is to deploy the new OS on Hawkbit. The FMU client
polls the server, detects an update query, and goes through `process_deployment()`. At this
point, it detects it is an OS update and will execute `update_system()`. This method does
multiple things : 
 1. Deploy the new deployment in OSTree
 1. **Delete  the `init_var` variable**. This will allow U-boot to start from scratch and
      re-initialize its variables.
 1. Write a JSON file stored in `/var/local/reboot_data.json` which typically looks like this : 
      ```json
      {
         "action_id":"24",
         "feedback_on_next_execution":true,
         "status_execution":1,
         "status_result":1,
         "msg":"OS fullmetalupdate-os-imx8mqevk v.202003190841 Deployment succeed"
      }
      ```
      This step is crucial as we need to store the information to **send the feedback on the next
      reboot**.

      `action_id` is the current deployment’s identifier. `feedback_on_next_execution` is a
      boolean used by the client after the reboot, here set to true. `status_execution` and
      `status_result` are two feedback statuses, and `msg` is the message sent to the server.

The client **does not give feedback yet**. Hence, the Hawkbit server considers the deployment
as pending, and waits for a feedback.

#### The U-boot script

The target reboots and executes the various command described above. Then the target
normally boots – or doesn’t, and executes the script again, incrementing the `trials`
variable. The target will then boot on either the new or the old deployment.

#### The FullMetalUpdate client

The client is executed at bootup with a systemd service. At initialization the client
executes `mark_os_successful()` which sets the U-boot `success` variable to 1 so that the
rollback conditions don’t execute on next reboot.

The client polls the server, and since the server is still in a deployment phase, the
client will execute `process_deployment()` again. Here, the client determines it’s an OS
update again, and will : 

 1. Execute `feedback_for_next_deployment()`. This method does a few important things : 
     - Load the json file previously written
     - Execute `check_for_rollback()`. This method checks if the target has rollbacked by looking
       if there is a pending deployment (i.e. the deployment which failed five times). If so, it
       undeploys it and returns `True` – and returns `False` otherwise. This method also rewrites
       `reboot_data.json` with `feedback_for_next_execution` to `False` for the next update.
     - Using `check_for_rollback()`'s value, change the result status and the message sent to the
       server to "failure" if the system has rollbacked
     - Return the feedback data
 1. Using `feedback_for_next_deployment()`'s return value, the client sees that
      `feedback_on_next_execution` is set to `true`, and feedbacks the server using the previously
      returned feedback data. It will also return from `process_deployment()` as we do not want to
      pursue the deployment process.

#### Notes : 

 - `feedback_for_next_deployment()` is executed at every execution of `process_deployment()`
 - If the target has never been updated before, `reboot_data.json` does not exist
     yet. `feedback_for_next_deployment()` handles this exception by returning a pseudo feedback
     data, containing only `feedback_for_next_execution` set to `False`. The file will be then
     written at the end of the deployment phase.
 - `reboot_data.json` is written in `/var` because this directory is not modified across
     deployments with OSTree

[ostree_comment_on_rollback]: https://github.com/ostreedev/ostree/issues/1808#issuecomment-459372967

## Rollbacking containers

### systemd's notify services

In FullMetalUpdate, all containers are started with systemd through the use of services.
The allows for proper life cycle, CGroups, and the usage of the great API systemd is offering
to manage services. Thus, runc containers are started with a [service file][sd_service_url]
which can be configured to start/stop the container and *execute commands after these
jobs*. This last point is important as it is the core of the FMU's rollback feature for
containers.

Systemd offers a type of service called `notify`. These services differ from `simple`
service because they are waiting for a signal coming from the main process (or any child
process depending on the configuration) before being in an activated mode. The service
will start the main process, and wait for this `notify` signal. If the service either
exits without sending the signal (successfully or unsuccessfully) or does not send the
signal during a *predefined delay*, the service is considered failed.

It is then handy to make use of that feature to determine if the container should rollback
to its previous version. As consequence the rollback feature for containers is
**optional**. In fact, you will have to adapt a container *recipes* and *service file* in
order to implement the functionality. This choice of being an optional feature rather than
a default, static feature mainly comes from the fact that containers may vary greatly
depending on the type of feature they implement. We will see now how to add the feature to
your recipes and service file.

Through this guide, we will simply name a "container that is started with a
systemd notify service" a **notify container** as it makes reading easier.

### How to adapt your container to use this feature

There are three different parts you'll need to adapt in order to use this feature :
 - your container application's source code (don't worry, it's rather simple)
 - the service file
 - the recipes

#### Adapting your source code

The only thing your source code will have to do is to send the `READY` flag to systemd.
There are currently two ways of doing that :
 - clang [`sd_notify()`][sd_notify_url] : `sd_notify(0, "READY=1")`
 - the shell command [`systemd-notify`][systemd-notify_url] : `systemd-notify --ready`

You should execute one of these commands at a strategic point in your program, when the
core features of your application have properly started. 

#### Adapting the service file

The following is an example service file using the rollback feature :
```
[Unit]
Description=Example container using systemd-notify
After=network.target

[Service]
Type=notify
NotifyAccess=all
WorkingDirectory=/apps/container-sd-notify/
ExecStart=/usr/bin/runc run container-sd-notify
ExecStartPost=sh /usr/fullmetalupdate/scripts/send_feedback.sh
ExecStopPost=sh /usr/fullmetalupdate/scripts/send_feedback.sh
TimeoutStartSec=35s
```

 - `Type=` must be set to `notify`
 - `NotifyAccess=` should be set to `all` if you want any of the child process of your main
     process to be able to send the `READY` flag
 - `ExecStartPost=` and `ExecStopPost=` will execute a script to send a message to the
     client. It **must** execute the `send_feedback.sh` script like in the example
 - `TimeoutStartSec=` is the delay your container has to send the `READY` flag to systemd.
     Set this delay depending on your application.

See [systemd's documentation][sd_service_url] for further details on these variables.

#### Adapt the recipes

The recipes will need `systemd` as part of its `IMAGE_INSTALL` variable.
Note that this will have no effect on the container's startup or behavior – it will just
install systemd's tools. `systemd` is needed in order to use [`sd_notify()`][sd_notify_url]
or [`systemd-notify`][systemd-notify_url]. 

The recipes will also need to set `NOTIFY` to `"1"`. This basically turns the notify
feature "on" for the container – under the assumption that you have properly configured it.
But also note that `AUTOSTART` and `TIMEOUT` are also **required variable** when `NOTIFY`
is set :
 - `AUTOSTART` should be set to `"1"`
 - `TIMEOUT` should be set to a delay value (in seconds), and it should be larger than
     `TimeoutStartSec=` in the service file. This delay is how much the client will
     ultimately wait until considering the deployment as failed (see the [How it
     works](#how-it-works) section)

### How it works

Communication between the client and the container service is made with a Unix socket.
There are multiple scenarios possible depending on the container execution, but we can
retain two results these executions can lead to :
 - the container executed properly, the notify flag is sent to systemd, and the client
     feedbacks the server positively
 - the container failed for some reason and the client considers the deployment as failed,
     leading to a rollback and a negative feedback

Let's first illustrate the client behavior with a successful deployment. We will then go
through the various reasons why a deployment could fail and see how the client handles it.

#### A successful notify container deployment

Once the container is [properly configured](#how-to-adapt-your-container-to-use-this-feature)
and built with Yocto, it needs to be deployed with Hawkbit. Let's go through the steps 
the target goes through :
 1. The client polls the server and notices an update is requested from the server. It
    also sets `action_id` to Hawkbit's action id, meaning the client is currently
    processing an update. As long as `action_id` is set the client will not download any
    other updates and process to any deployment
 1. The client reads the metadata included in the distribution. It will store `notify` (`1`
    when the distribution implements the rollback feature, `0` otherwise) and `timeout`,
    the delay [mentioned previously](#adapt-the-recipes). These are the two variables set
    in the recipes.
 1. Since `notify` is set to `1`, it does two things :
     - it creates a Unix socket that will be used to receive a message from systemd
     - it **starts a thread** and passes the information about the deployment (the socket,
         the revision sha, autostart, autoremove…) to the thread. This thread will wait on
         the socket for an incoming message from systemd
 1. The client triggers the update : service is stopped if pre-existing, data is pulled
    from the remote repo, container is checked out to the new revision, and the service is
    started again
 1. The service starts the main process running in the container
 1. The process initializes and sends the `READY` flag to systemd
 1. Systemd receives the flag and executes the `ExecStartPost=` command, that will itself
    execute the `send_feedback.sh` script
 1. This script will simply send `success` to the container using `socat`
 1. The client's waiting thread receives the data and parses it. Since `success` is
    received, the client will feedback the server positively.
 1. The thread will also write the revision sha to a json file. This is important because
    OSTree's API and behavior don't allow us to get the current revision where the container
    is checked out. This stored revision will be used in case of failure on another
    update
 1. The thread unsets `action_id` and the main client process can process other updates

#### Handling failure and rollbacking

##### How the container may fail

The container may fail for multiple reasons :
 - **Case 1** : the container exits successfully without sending the `READY` flag
 - **Case 2** : the container fails before sending the `READY` flag
 - **Case 3** : the container does not send `READY` before the timeout set in
     `TimeoutStartSec=`
 - **Case 4** : the socket communication fails and the message is not received by the
     client's thread

In the first three cases, `ExecStopPost=` is executed. The command executed in
`ExecStopPost=` is given `SERVICE_RESULT`, `EXIT_CODE` and `EXIT_STATUS` as environment
variables, whereas `ExecStartPost=` does not give any of these – this is how we
differentiate the two commands and execute the same script.

For the first three cases, the results are sent to Hawkbit. This is what they look like :

|            | `SERVICE_RESULT` | `EXIT_CODE` | `EXIT_RESULT` |
|:----------:|:----------------:|:-----------:|:-------------:|
| **Case 1** |    `protocol`    |   `exited`  |       0       |
| **Case 2** |    `exit-code`   |   `exited`  |       1       |
| **Case 3** |     `timeout`    |   `exited`  |      143      |
 
##### How the client handles a failure

When the container fails, `ExecStopPost=` is always executed. It will send the three
environment variables to the client through the socket.

When the client's thread receives the data from systemd, it parses it and determines the
service has failed. At this point, the thread will read into the json file mentioned
before to get the successful revision sha, and update the container to this revision — in
other words, **rollback**. If this update fails the client will let the server know about
it. This can happen when this is the first time installing the container.

In the case where the client has not received the datagram even after the
`TimeoutStartSec=` delay, the socket times out after the `TIMEOUT` recipes' delay. This
would not happen often, but in this case the client will still rollback the container and
let the server know that the socket timed out.

[sd_service_url]: https://www.freedesktop.org/software/systemd/man/systemd.service.html
[sd_notify_url]: https://www.freedesktop.org/software/systemd/man/sd_notify.html
[systemd-notify_url]: https://www.freedesktop.org/software/systemd/man/systemd-notify.html
