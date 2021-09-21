
# Procedure: Node Replace

### Pre-Requisites

<Check cluster health?>

### Node start

[Starting a node](https://www.cockroachlabs.com/docs/stable/cockroach-start.html)

### Node stop

There are multiple ways to stop a CRDB node based on how it was started:

- If the node was started with a process manager, gracefully stop the node by sending SIGTERM with the process manager. If the node is not shutting down after 1 minute, send SIGKILL to terminate the process. When using systemd, for example, set TimeoutStopSecs=60 in your configuration template and run systemctl stop \&lt;systemd config filename\&gt; to stop the node without systemd restarting it.
- If the node was started using cockroach start and is running in the foreground, press ctrl-c in the terminal.
- If the node was started using cockroach start and the _--background_ and -- **pid** - **file** flags, run kill \&lt;pid\&gt;, where **\&lt;pid\&gt;** is the process ID of the node.

[TODO:

Remove the node from LB

5 minutes minimum to allow a graceful exit of nodes… drain connections, move leases, etc.
 There is a setting in systemd conf file and k8s - include here
 TimeoutStopSecs should be 300, not 60, to ensure orderly node draining and orderly shutdown]

### Node restart

Restarting a node requires that you first stop the node from the previous step and then start the node again following the instructions from 3.2.1.

