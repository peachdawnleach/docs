You can use `cockroach cert` commands or [`openssl` commands]({% link {{ page.version.version }}/create-security-certificates-openssl.md %}) to generate security certificates. This section features the `cockroach cert` commands.

Locally, you'll need to [create the following certificates and keys]({% link {{ page.version.version }}/cockroach-cert.md %}):

- A certificate authority (CA) key pair (`ca.crt` and `ca.key`).
- A node key pair for each node, issued to its IP addresses and any common names the machine uses, as well as to the IP addresses and common names for machines running load balancers.
- A client key pair for the `root` user. You'll use this to run a sample workload against the cluster as well as some `cockroach` client commands from your local machine.

{{site.data.alerts.callout_success}}Before beginning, it's useful to collect each of your machine's internal and external IP addresses, as well as any server names you want to issue certificates for.{{site.data.alerts.end}}

1. [Install CockroachDB]({% link {{ page.version.version }}/install-cockroachdb.md %}) on your local machine, if you haven't already.

1. Create two directories:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ mkdir certs
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ mkdir my-safe-directory
    ~~~
    - `certs`: You'll generate your CA certificate and all node and client certificates and keys in this directory and then upload some of the files to your nodes.
    - `my-safe-directory`: You'll generate your CA key in this directory and then reference the key when generating node and client certificates. After that, you'll keep the key safe and secret; you will not upload it to your nodes.

1. Create the CA certificate and key:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-ca \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key
    ~~~

1. Create the certificate and key for the first node, issued to all common names you might use to refer to the node as well as to the load balancer instances:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-node \
    <node1 internal IP address> \
    <node1 external IP address> \
    <node1 hostname>  \
    <other common names for node1> \
    localhost \
    127.0.0.1 \
    <load balancer IP address> \
    <load balancer hostname>  \
    <other common names for load balancer instances> \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key
    ~~~

1. Upload the CA certificate and node certificate and key to the first node:
   
    {% if page.title contains "Google" %}
    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ gcloud compute ssh \
    --project <project name> <instance name> \
    --command "mkdir certs"
    ~~~

    {{site.data.alerts.callout_info}}
    `gcloud compute ssh` associates your public SSH key with the GCP project and is only needed when connecting to the first node. See the [GCP docs](https://cloud.google.com/sdk/gcloud/reference/compute/ssh) for more details.
    {{site.data.alerts.end}}

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ scp certs/ca.crt \
    certs/node.crt \
    certs/node.key \
    <username>@<node1 address>:~/certs
    ~~~

    {% elsif page.title contains "AWS" %}
    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ ssh-add /path/<key file>.pem
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ ssh <username>@<node1 DNS name> "mkdir certs"
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ scp certs/ca.crt \
    certs/node.crt \
    certs/node.key \
    <username>@<node1 DNS name>:~/certs
    ~~~

    {% else %}
    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ ssh <username>@<node1 address> "mkdir certs"
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ scp certs/ca.crt \
    certs/node.crt \
    certs/node.key \
    <username>@<node1 address>:~/certs
    ~~~
    {% endif %}

1. Delete the local copy of the node certificate and key:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ rm certs/node.crt certs/node.key
    ~~~

    {{site.data.alerts.callout_info}}
    This is necessary because the certificates and keys for additional nodes will also be named `node.crt` and `node.key`. As an alternative to deleting these files, you can run the next `cockroach cert create-node` commands with the `--overwrite` flag.
    {{site.data.alerts.end}}

1. Create the certificate and key for the second node, issued to all common names you might use to refer to the node as well as to the load balancer instances:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-node \
    <node2 internal IP address> \
    <node2 external IP address> \
    <node2 hostname>  \
    <other common names for node2> \
    localhost \
    127.0.0.1 \
    <load balancer IP address> \
    <load balancer hostname>  \
    <other common names for load balancer instances> \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key
    ~~~

1. Upload the CA certificate and node certificate and key to the second node:

    {% if page.title contains "AWS" %}
    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ ssh <username>@<node2 DNS name> "mkdir certs"
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ scp certs/ca.crt \
    certs/node.crt \
    certs/node.key \
    <username>@<node2 DNS name>:~/certs
    ~~~

    {% else %}
    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ ssh <username>@<node2 address> "mkdir certs"
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ scp certs/ca.crt \
    certs/node.crt \
    certs/node.key \
    <username>@<node2 address>:~/certs
    ~~~
    {% endif %}

1. Repeat steps 6 - 8 for each additional node.

1. Create a client certificate and key for the `root` user:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach cert create-client \
    root \
    --certs-dir=certs \
    --ca-key=my-safe-directory/ca.key
    ~~~

1. Upload the CA certificate and client certificate and key to the machine where you will run a sample workload:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ ssh <username>@<workload address> "mkdir certs"
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ scp certs/ca.crt \
    certs/client.root.crt \
    certs/client.root.key \
    <username>@<workload address>:~/certs
    ~~~

    In later steps, you'll also use the `root` user's certificate to run [`cockroach`]({% link {{ page.version.version }}/cockroach-commands.md %}) client commands from your local machine. If you might also want to run `cockroach` client commands directly on a node (e.g., for local debugging), you'll need to copy the `root` user's certificate and key to that node as well.

{{site.data.alerts.callout_info}}
On accessing the DB Console in a later step, your browser will consider the CockroachDB-created certificate invalid and you’ll need to click through a warning message to get to the UI. You can avoid this issue by [using a certificate issued by a public CA]({% link {{ page.version.version }}/create-security-certificates-custom-ca.md %}#accessing-the-db-console-for-a-secure-cluster).
{{site.data.alerts.end}}
