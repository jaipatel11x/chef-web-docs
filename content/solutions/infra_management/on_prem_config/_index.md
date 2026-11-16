+++
title = "The Chef way of on-prem configuration management"
description = "Manage your on-premises server fleet with Chef: standardize configuration across servers, correct drift automatically, and maintain consistency without manual configuration."
draft = false
gh_repo = "chef-web-docs"

[menu.overview]
  identifier = "overview/solutions/infra_management/on_prem_config"
  parent = "overview/solutions/infra_management"
  title = "On-prem config management"
  weight = 30
+++

<!-- cspell:ignore ntp sysctl ntpd chronyd converge idempotent -->

Your on-premises infrastructure drifts when servers are configured by hand.
One administrator uses a different SSH key algorithm than another.
An incident causes a temporary change that becomes permanent.
A new server is built from outdated documentation.
A security scan finds that one server is missing a critical hardening step.

Chef solves this by letting you describe your infrastructure as code and by running Chef Infra Client on each server to keep it matching that code.
The agent changes only what has drifted, so manual changes don't survive for long, and every change is logged and reported.

This page covers how Chef Infra Client manages on-premises server fleets at scale.
Chef Infra Client doesn't handle bare-metal provisioning, doesn't manage the network, and doesn't back up your servers.
See [Where Chef fits](#where-chef-fits) for the full picture.

## Who this page is for

**If you're accountable for infrastructure consistency**, read this section and [Where Chef fits](#where-chef-fits).
You'll learn what Chef gives you, what it doesn't, and how it keeps servers in a known state.

**If you manage the servers**, work through the rest of the page.
It builds a small cookbook, bootstraps servers, shows how drift is detected and corrected, and integrates compliance auditing.

### What you get

- **Consistency.** Every server of a given type is built from the same code.
- **Drift correction.** Configuration drift is detected and corrected automatically.
- **Multi-server orchestration.** A single policy applies to dozens or hundreds of servers.
- **Staged rollouts.** Test a change on a few servers, then roll it out to your entire fleet.
- **Compliance enforcement.** Run [Chef InSpec](/inspec/latest/) audits as part of each Chef Infra Client run.
- **Auditability.** Every run reports which resources changed and when.

### What you don't get

Chef Infra Client configures the servers it runs on.
It doesn't provision bare-metal systems, manage network configuration, back up data, or know when it's safe to restart a critical service.
A server is in a known state only for the settings you wrote code for and only as of its last successful Chef Infra Client run.

## Where Chef fits

On-premises infrastructure consists of multiple layers.

| Layer | What it means | Chef piece | On this page |
| --- | --- | --- | --- |
| Bare-metal provisioning | Booting servers from scratch, partitioning disks, installing the OS | Provisioning tools like Cobbler or MAAS | No |
| Operating system baseline | Packages, kernel parameters, users, file permissions, services | [Chef Infra Client](/client/latest/) resources | Yes |
| Application configuration | Runtime versions, dependencies, application-specific settings | [Chef Infra cookbooks](/client/latest/cookbooks/) | Yes |
| Credentials and secrets | Passwords, API keys, certificates | [Chef Vault](/client/latest/cookbook_patterns/data_bags/) or external secrets services | See [Secrets and credentials](/client/latest/infra_language/secrets/) |
| Compliance and auditing | Verifying systems meet security and regulatory requirements | [Chef InSpec](/inspec/latest/) | See [Integrate compliance](#integrate-compliance) |
| Monitoring and reporting | Understanding the state and health of your infrastructure | [Chef Automate](/automate/client_runs/) | See [Visibility and reporting](#visibility-and-reporting) |

## What you'll build

A Chef Infra cookbook, `onprem_baseline`, that manages these core settings on a Linux server:

- Installs and configures NTP for system time synchronization.
- Manages SSH configuration for security.
- Sets kernel parameters for performance.
- Validates configuration after each change.

You'll bootstrap a test server, run Chef Infra Client, make a manual change to show how drift is detected, and watch Chef correct it.

## Before you begin

You need:

- [Chef Workstation](/workstation/latest/install/) with the `chef` and `chef-client` commands.
- A Chef license key.
  See [Chef license key](/licensing/license_key/).
- [Chef 360 Platform Declarative State Management (DSM)](/360/latest/understand/dsm/) or [Chef Infra Server](/server/) accessible from your on-premises network.
- One or more Linux test servers (physical or virtual) running Ubuntu 22.04 or later.
  Don't run this on production servers.
- SSH access to the test servers with `sudo` privilege.

Output on this page comes from Chef Infra Client 19.4 running on Ubuntu 22.04.
Output from a different release or distribution might differ slightly.

## Write the cookbook

Create a repository and a cookbook:

```shell
chef generate repo chef-repo
cd chef-repo
chef generate cookbook cookbooks/onprem_baseline
```

Create `cookbooks/onprem_baseline/attributes/default.rb`:

```ruby
default['onprem_baseline']['ntp_servers'] = [
  '0.pool.ntp.org',
  '1.pool.ntp.org',
  '2.pool.ntp.org',
  '3.pool.ntp.org',
]

default['onprem_baseline']['ssh_config'] = {
  'PermitRootLogin' => 'no',
  'PasswordAuthentication' => 'no',
  'MaxAuthTries' => '3',
  'LogLevel' => 'VERBOSE',
}

default['onprem_baseline']['kernel_params'] = {
  'net.ipv4.tcp_syncookies' => '1',
  'net.ipv4.tcp_max_syn_backlog' => '2048',
  'net.core.somaxconn' => '1024',
}
```

Create `cookbooks/onprem_baseline/recipes/default.rb`:

```ruby
package 'chrony' do
  action :install
end

file '/etc/chrony/chrony.conf.d/onprem-baseline.conf' do
  content lazy {
    node['onprem_baseline']['ntp_servers'].map { |server| "server #{server} iburst" }.join("\n") + "\n"
  }
  notifies :restart, 'service[chrony]'
end

service 'chrony' do
  action [:enable, :start]
end

file '/etc/ssh/sshd_config.d/00-onprem-baseline.conf' do
  content lazy {
    node['onprem_baseline']['ssh_config'].map { |key, value| "#{key} #{value}" }.join("\n") + "\n"
  }
  notifies :run, 'execute[validate-sshd-config]', :immediately
  notifies :restart, 'service[ssh]'
end

execute 'validate-sshd-config' do
  command 'sshd -t'
  action :nothing
end

service 'ssh' do
  action [:enable, :start]
end

node['onprem_baseline']['kernel_params'].each do |param, value|
  sysctl param do
    value value
    action :apply
  end
end
```

Update `cookbooks/onprem_baseline/metadata.rb`:

```ruby
name 'onprem_baseline'
description 'Manage on-premises server baseline configuration'
license 'Apache-2.0'
version '0.1.0'

supports 'ubuntu', '>= 22.04'
supports 'redhat', '>= 8.0'
```

## Bootstrap servers to Chef

Bootstrapping registers a server with Chef and installs Chef Infra Client.

```bash
knife bootstrap <SERVER_IP_ADDRESS> \
  --chef-license accept-silent \
  --run-list 'recipe[onprem_baseline]' \
  --policy-name onprem-prod \
  --policy-group production \
  --user ubuntu \
  --sudo
```

This command connects via SSH, installs Chef Infra Client, registers with Chef 360 Platform DSM, and runs the first converge.

## Converge a server and correct drift

Chef Infra Client runs on a schedule (every 30 minutes by default).
It fetches the latest policies and cookbooks, compares the actual server state against the desired state, and updates any resources that don't match.

To trigger a run immediately:

```bash
ssh ubuntu@<SERVER_IP_ADDRESS> sudo chef-client
```

### Output from a converged server

All settings already match:

```text
[2024-11-16T10:35:00+00:00] INFO: Chef Infra Client, version 19.4.46
Chef Infra Client Infra Phase complete, 0/5 resources updated in 2.14s
```

When nothing changed, Chef Infra Client is idempotent: running it again produces no changes.

### When drift occurs

An administrator manually changes the SSH configuration:

```bash
ssh ubuntu@server1.onprem.local sudo sed -i 's/MaxAuthTries 3/MaxAuthTries 5/' /etc/ssh/sshd_config.d/00-onprem-baseline.conf
```

On the next Chef Infra Client run, drift is corrected:

```text
[2024-11-16T11:00:00+00:00] INFO: Chef Infra Client, version 19.4.46
[2024-11-16T11:00:02+00:00] INFO: Processing file[/etc/ssh/sshd_config.d/00-onprem-baseline.conf] action create
[2024-11-16T11:00:02+00:00] INFO: file[/etc/ssh/sshd_config.d/00-onprem-baseline.conf] updated content
Chef Infra Client Infra Phase complete, 2/5 resources updated in 4.32s
```

The file is reverted, and the server is back in the known state.

## Update configuration across your fleet

To update configuration on all servers, change the cookbook and push it to Chef 360 Platform DSM.

For example, to change the NTP pool servers, edit `attributes/default.rb`:

```ruby
default['onprem_baseline']['ntp_servers'] = [
  '0.pool.ntp.org',
  '1.pool.ntp.org',
  'custom-ntp.internal',
]
```

Then upload the updated cookbook:

```bash
cd chef-repo
knife cookbook upload cookbooks/onprem_baseline --use-current-user
```

On the next Chef Infra Client run, all servers with the `onprem-prod` policy will download the updated cookbook and converge.

For multi-server orchestration, use [Policyfiles](/client/latest/policy/policyfile/) to version multiple cookbooks together:

```ruby
# chef-repo/Policyfile.rb
name 'onprem-prod'
run_list 'recipe[onprem_baseline]'
cookbook 'onprem_baseline', '0.1.0'
```

Upload the policy:

```bash
knife policy push production Policyfile.rb
```

## Use different settings for different environments

Different server groups often need different settings.
Production servers might have more NTP servers than development ones.
Different environments might have different SSH logging levels.

Use [attributes](/client/latest/cookbooks/attributes/) to manage these differences without duplicating cookbooks.

**Define environment attributes in Chef 360 Platform DSM:**

```ruby
# Production environment
{
  "onprem_baseline" => {
    "ntp_servers" => ["0.pool.ntp.org", "1.pool.ntp.org", "2.pool.ntp.org", "3.pool.ntp.org"],
  }
}

# Development environment
{
  "onprem_baseline" => {
    "ntp_servers" => ["0.pool.ntp.org", "1.pool.ntp.org"],
  }
}
```

Then assign servers to environments during bootstrap:

```bash
knife bootstrap <DEV_SERVER> --policy-name onprem-dev --policy-group development
knife bootstrap <PROD_SERVER> --policy-name onprem-prod --policy-group production
```

Each group receives its own attribute values.

## Integrate compliance

Configuration management and compliance checking work together.
Once your servers are converged to a known state, audit them to verify they meet security and regulatory requirements.

Chef Infra Client supports the [Compliance Phase](/client/latest/features/chef_compliance_phase/), which runs [Chef InSpec](/inspec/latest/) audits after converging.

Enable the Compliance Phase in your policy:

```ruby
# chef-repo/Policyfile.rb
name 'onprem-prod'
run_list 'recipe[onprem_baseline]'

audit_cookbook_version '7.9.0'
run_list_audit 'recipe[audit::default]'
```

Create a Chef InSpec profile:

```ruby
# cookbooks/onprem_baseline/spec/controls/baseline.rb
control 'onprem-ntp-1' do
  title 'NTP service is running and enabled'
  describe service('chrony') do
    it { should be_running }
    it { should be_enabled }
  end
end

control 'onprem-ssh-1' do
  title 'SSH root login is disabled'
  describe file('/etc/ssh/sshd_config.d/00-onprem-baseline.conf') do
    its('content') { should match(/^PermitRootLogin no$/) }
  end
end
```

On each Chef Infra Client run, the profile is executed after convergence.
If a resource is out of spec, it shows as a failed audit in Chef Automate.

See [The Chef way of compliance](/solutions/compliance/) for deeper guidance on building audits.

## Visibility and reporting

As your on-premises infrastructure scales, you need visibility into what's happening across your fleet.

Chef Automate provides dashboards showing run status, resources that changed, nodes that are out of compliance, and customizable reports for auditors.

Configure Chef Infra Client to report to Chef Automate:

```bash
knife bootstrap <SERVER> \
  --chef-server-url https://dsm.internal/organizations/default \
  --automate-server-url https://automate.internal \
  --policy-name onprem-prod \
  --policy-group production
```

Nodes will send run results to both Chef 360 Platform DSM and Chef Automate automatically.

## Supported platforms

Chef Infra Client supports most Linux distributions and Windows.
The examples on this page use Ubuntu 22.04 with `apt` and systemd.

For the full list of supported platforms and system requirements, see:

- [Chef Infra Client supported platforms](/platforms/#chef-infra-client-support)
- [Chef Infra Client system requirements](/client/latest/install/system_requirements/)

## Next steps

- **Write your first cookbook:** Follow the [Chef Infra quickstart guide](/client/latest/quickstart/)
- **Learn about Policyfiles:** Use [Policyfiles](/client/latest/policy/policyfile/) to version and promote cookbooks through environments
- **Bootstrap nodes:** See [Bootstrap servers](/client/latest/install/bootstrap/) for detailed bootstrapping options
- **Monitor compliance:** Set up [Chef InSpec audits](/inspec/latest/) and [Compliance Phase](/client/latest/features/chef_compliance_phase/) integration
- **Use Chef Automate:** Use [Chef Automate](/automate/client_runs/) for fleet-wide visibility and reporting
- **Understand Chef 360 Platform:** Explore [Chef 360 Platform DSM](/360/latest/understand/dsm/) for centralized policy management
- **Migrate from Chef Infra Server:** See [Migrate to Chef 360 Platform](/360/latest/migrate/on_prem_to_self_managed/)
