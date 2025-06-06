---
title: CockroachDB Security Overview
summary: An comparison of CockroachDB Cloud's Security Features
toc: true
docs_area: reference.security
---
## Ways to Use CockroachDB

### CockroachDB Cloud

CockroachDB {{ site.data.products.cloud }} provides fast and easy access (including a *free* tier) to CockroachDB as a web service, hosted by Cockroach Labs. Clusters run in multi-tenant Google Cloud Platform (GCP) or Amazon Web Services (AWS) environments with shared compute and networking resources.

CockroachDB {{ site.data.products.advanced }} offers a single-tenant cluster running in its own Virtual Private Cloud (VPC). Compute and networking resources are isolated. CockroachDB {{ site.data.products.advanced }} provides additional security-enhancing features such as single sign-on (SSO) and SQL audit logging.

[Sign up for a CockroachDB Cloud account!](https://www.cockroachlabs.com/get-started-cockroachdb/)

{% include {{ page.version.version }}/dedicated-pci-compliance.md %}

Learn more: [Integrate CockroachDB {{ site.data.products.advanced }} with Satori]({% link {{ page.version.version }}/satori-integration.md %})

### Self-Hosted

CockroachDB {{ site.data.products.core }} here refers to the situation of a user deploying and operating their own cluster.

{{ site.data.products.enterprise }} refers to an ongoing license relationship with Cockroach Labs. In this situation the customer maintains full control over their data, compute, and network resources while benefiting from the expertise of Cockroach Labs' {{ site.data.products.enterprise }} Support staff.

For more information, see the [licensing FAQ]({% link {{ page.version.version }}/licensing-faqs.md %})

## Comparison of security features

<table>
  <thead>
    <tr>
      <td width="100">Security Domain</td>
      <td>{{ site.data.products.basic }}</td>
      <td>{{ site.data.products.standard }}</td>
      <td>{{ site.data.products.advanced }}</td>
      <td>{{ site.data.products.core }} {{ site.data.products.enterprise }}</td>
      <td width="120">Feature</td>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan="8"><a href="authentication.html">Authentication</a></td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>Inter-node and node identity authentication using TLS 1.3</td>
    </tr>
    <tr>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>Client identity authentication using username/password</td>
    </tr>
    <tr>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td><a href="scram-authentication.html">SASL/SCRAM-SHA-256 secure password-based authentication</a></td>
    </tr>
    <tr>
      <td>&nbsp;</td>
      <td>&nbsp;</td>
      <td>&nbsp;</td>
      <td>✓</td>
      <td>SQL client identity authentication using TLS 1.2/1.3</td>
    </tr>
    <tr>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>Web console authentication with third-party <a href="{% link {{ page.version.version }}/sso-db-console.md %}">Single Sign-on (SSO)</a> using <a href="https://openid.net/connect/">OpenID Connect OIDC</a></td>
    </tr>
    <tr>
      <td>&nbsp;</td>
      <td>&nbsp;</td>
      <td>&nbsp;</td>
      <td>✓</td>
      <td>Client identity authentication with <a href="{% link {{ page.version.version }}/gssapi_authentication.md %}">GSSAPI and Kerberos</a></td>
    </tr>
    <tr>
      <td>&nbsp;</td>
      <td>&nbsp;</td>
      <td>&nbsp;</td>
      <td>✓</td>
      <td>HTTP API access using login tokens</td>
    </tr>
    <tr>
      <td>&nbsp;</td>
      <td>&nbsp;</td>
      <td>&nbsp;</td>
      <td>✓</td>
      <td><a href="https://wikipedia.org/wiki/Online_Certificate_Status_Protocol">OCSP</a> certificate revocation protocol</td>
    </tr>
    <tr>
      <td rowspan="5" ><a href="encryption.html">Encryption</a></td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>Encryption in transit using TLS 1.3</td>
    </tr>
    <tr>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>Backups for AWS clusters are encrypted at rest using <a href="https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingServerSideEncryption.html">AWS S3’s server-side encryption</a></td>
    </tr>
    <tr>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>Backups for GCP clusters are encrypted at rest using <a href="https://cloud.google.com/storage/docs/encryption/default-keys">Google-managed server-side encryption keys</a></td>
    </tr>
    <tr>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>Industry-standard encryption at rest is provided at the infrastructure level by your chosen deployment environment, such as Google Cloud Platform (GCP), Amazon Web Services (AWS), or Microsoft Azure. You can learn more about <a href="https://cloud.google.com/compute/docs/disks#pd_encryption">GCP persistent disk encryption</a>, <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSEncryption.html">AWS Elastic Block Storage</a>, or <a href="https://docs.microsoft.com/azure/virtual-machines/disk-encryption-overview">Azure managed disk encryption</a>.
    </tr>
    <tr>
      <td>&nbsp;</td>
      <td>&nbsp;</td>
      <td>&nbsp;</td>
      <td>✓</td>
      <td>Cockroach Labs's proprietary storage-level <a href="encryption.html#encryption-at-rest-enterprise">{{ site.data.products.enterprise }} Encryption At Rest service</a> implementing the <a href="https://wikipedia.org/wiki/Advanced_Encryption_Standard">Advanced Encryption Standard (AES)</a></td>
    </tr>
    <tr>
      <td rowspan="2" ><a href="authorization.html">Authorization</a></td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>Users and privileges</td>
    </tr>
    <tr>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>Role-based access control (RBAC)</td>
    </tr>
    <tr>
      <td rowspan="3">Network Security</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td><a href="authentication.html"></a>SQL-level configuration allowed authentication attempts by IP address</td>
    </tr>
    <tr>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>Network-level Configuration of allowed IP addresses</td>
    </tr>
    <tr>
      <td>&nbsp;</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td><a href="{% link cockroachcloud/connect-to-your-cluster.md %}#gcp-private-service-connect">GCP Private Service Connect (PSC)</a> or <a href="{% link cockroachcloud/connect-to-your-cluster.md %}#vpc-peering">VPC Peering</a> for GCP clusters and <a href="{% link cockroachcloud/aws-privatelink.md %}">AWS PrivateLink</a> for AWS clusters </td>
    </tr>
    <tr>
      <td><a href="https://wikipedia.org/wiki/Non-repudiation">Non-Repudiation</a></td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td><a href="{% link {{ page.version.version }}/sql-audit-logging.md %}">SQL Audit Logging</a></td>
    </tr>
    <tr>
      <td><a href="{% link {{ page.version.version }}/demo-fault-tolerance-and-recovery.md %}">Availability/Resilience</a></td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>✓</td>
      <td>CockroachDB, as a distributed SQL database, is uniquely resilient by nature. A cluster can tolerate node failures as long as the majority of nodes remain functional. See <a href="{% link {{ page.version.version }}/demo-fault-tolerance-and-recovery.md %}">Disaster Recovery.</a></td>
    </tr>
  </tbody>
</table>
