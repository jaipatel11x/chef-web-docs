+++
title = "The Chef way of on-prem configuration management"
description = "Manage your on-premises server fleet with Chef: standardize configuration across dozens or hundreds of servers, correct drift automatically, enforce compliance, and maintain consistency without manual configuration."
draft = false
gh_repo = "chef-web-docs"

[menu.overview]
  identifier = "overview/solutions/infra_management/on_prem_config"
  parent = "overview/solutions/infra_management"
  title = "On-prem config management"
  weight = 30
+++

<!-- cspell:ignore ntp sysctl ntpd chronyd converge idempotent -->

Your on-premises infrastructure is built on consistency.
Every web server should have the same load balancing configuration.
Every database server should have the same security settings and kernel tuning.
Every application server should have the same runtime version and dependencies.
When servers are configured by hand, they drift apart.
One administrator uses a different SSH key algorithm than another.
An incident causes a temporary change that becomes permanent.
A new server is built from outdated documentation, and the difference doesn't surface until a performance problem appears.
A security scan finds that one server is missing a critical hardening step.

You need one reviewed definition of how every server should be configured, and a way to keep every server matching it, automatically, across dozens or hundreds of systems.
Chef does this by letting you describe your infrastructure as code and by running an agent, Chef Infra Client, on each server that makes the server match that code, repeatedly and reliably.
The agent changes only what has drifted, so manual changes don't survive long, and every change is logged and reported.

This page covers how Chef Infra Client and [Chef 360 Platform](/360/latest/) work together to manage on-premises server fleets at scale.
It's honest about the limits: Chef Infra Client doesn't handle bare-metal provisioning, doesn't manage the network, and doesn't back up your servers.
See [Where Chef fits](#where-chef-fits) for the full picture.

## Who this page is for

**If you're accountable for the outcome**, such as an infrastructure or platform owner, read this section and [Where Chef fits](#where-chef-fits).
You'll learn what Chef gives you for server fleet consistency, what it doesn't, and how the architecture keeps servers in a known state.

**If you manage the servers**, work through the rest of the page.
It builds a small cookbook that manages core settings on a fleet of servers, bootstraps servers to Chef, deploys configuration through roles and policies, shows how drift is detected and corrected automatically, and integrates compliance auditing.
If you haven't used Chef before, you don't need to read anything else first.

### What you get

- **Consistency across your fleet.** Every server of a given type is built from the same code, so servers can't quietly drift apart.
- **Automatic drift correction.** When a configuration setting changes, Chef Infra Client reverts it on the next run and logs what it changed.
- **Multi-server orchestration.** A single policy applies to dozens or hundreds of servers, and you can target groups by role or environment.
- **Staged rollouts.** Test a configuration change on a few servers, verify it works, then roll it out to the entire fleet.
- **Compliance enforcement.** Run [Chef InSpec](/inspec/latest/) assessments as part of each Chef Infra Client run to verify that systems meet security and operational requirements.
- **Detailed auditability.** Every run reports which resources changed, when they changed, and by whom.
- **Flexible deployment.** Run Chef Infra Client as a service on each node, as a scheduled job, or through Chef 360 Platform job automation.

### What you don't get

Chef Infra Client configures the servers it runs on.
It doesn't provision bare-metal systems, doesn't manage network configuration at the switch level, doesn't back up data, and doesn't know when it's safe to restart a critical service.
A server is in a known state only for the settings you wrote code for, and only as of its last successful Chef Infra Client run.
Configuration drift can occur between runs if someone makes a manual change.

## Where Chef fits

On-premises infrastructure consists of multiple layers, and Chef Infra Client covers the most critical one: the server configuration itself.
Understanding where Chef fits helps you plan how to use it effectively.

| Layer | What it means | Chef piece | On this page |
| --- | --- | --- | --- |
| Bare-metal provisioning | Booting servers from scratch, partitioning disks, installing the OS | Provisioning tools like Cobbler, MAAS, or cloud-init scripts | No |
| Operating system baseline | Packages, kernel parameters, users, file permissions, services | [Chef Infra Client](/client/latest/) resources such as `package`, `sysctl`, `service`, `user`, and `file` | Yes |
| Application configuration | Runtime versions, dependencies, application-specific settings | [Templates](/client/latest/resources/bundled/template/) and [Chef Infra cookbooks](/client/latest/cookbooks/) | Yes |
| Credentials and secrets | Passwords, API keys, certificates | [Data Bags](/server/api_data_bags/), [Chef Vault](/client/latest/cookbook_patterns/data_bags/), or external secrets services | Use templates and node attributes |
| Compliance and auditing | Verifying that systems meet security and regulatory requirements | [Chef InSpec](/inspec/latest/) assessments run during the [Compliance Phase](/client/latest/features/chef_compliance_phase/) | See [Integrate compliance](#integrate-compliance) |
| Monitoring and reporting | Understanding the state and health of your infrastructure | [Chef Automate](/automate/client_runs/) and reporting integrations | See [Visibility and reporting](#visibility-and-reporting) |

The core Chef Infra workflow for on-premises infrastructure:

| Step | What it does | Chef component |
| --- | --- | --- |
| Define configuration as code | You write recipes and cookbooks that describe how servers should be configured | [Chef Infra cookbooks](/client/latest/cookbooks/) and [resources](/client/latest/resources/) |
| Store policies centrally | Cookbooks and policies are stored in a central server that all nodes can reach | [Chef 360 Platform Declarative State Management (DSM)](/360/latest/understand/dsm/) or [Chef Infra Server](/server/) |
| Enroll nodes | Each server runs Chef Infra Client, which registers with the central server and downloads its cookbooks | [Bootstrap](/client/latest/install/bootstrap/) or [enrollment](/360/latest/use/operators/node_enrollment/) |
| Converge repeatedly | Chef Infra Client runs on a schedule (typically every 30 minutes) and updates the server to match the code | [Chef Infra Client](/client/latest/) daemon or cron job |
| Detect drift | Drift is detected automatically when resources don't match the desired state | Built into Chef Infra Client |
| Correct drift | Out-of-spec resources are updated automatically without manual intervention | Idempotent resource execution |
| Report results | Each run sends back logs and resource-level details of what changed | Sent to central server and [Chef Automate](/automate/client_runs/) |

## What you'll build

A Chef Infra cookbook, `onprem_baseline`, that does these things on a Linux server:

- Installs and configures NTP to keep system time synchronized.
- Manages core SSH configuration for security and consistency.
- Sets kernel parameters for performance and stability.
- Validates the configuration after each change.

It's a stand-in for real work, but the same pattern applies to any on-premises server: manage packages, configuration files, and services as code.
You'll bootstrap a test server, run Chef Infra Client, make a manual change to show how drift is detected, and then watch Chef correct it.

For multi-server orchestration, you'll learn how to assign roles and policies so the same cookbook applies to your entire fleet with environment-specific differences.

## Before you begin

You need:

- [Chef Workstation](/workstation/latest/install/) on the machine where you write and test code.
  It includes the `chef` and `chef-client` commands.
- A Chef license key to bootstrap nodes and use Chef 360 Platform.
  See [Chef license key](/licensing/license_key/).
- A progress from Chef 360 Platform [Declarative State Management (DSM)](/360/latest/understand/dsm/) or a [Chef Infra Server](/server/) instance accessible from your on-premises network.
- One or more Linux test servers (physical or virtual).
  The examples use Ubuntu 22.04 or later, with `apt` package management and systemd for service management.
  Don't run this on production servers.
- SSH access to the test servers with `sudo` privilege.

The output on this page comes from Chef Infra Client 19.4 running on Ubuntu 22.04 servers.
Output from a different release or distribution might differ slightly.

## Write the cookbook

To create a cookbook that manages your on-premises server baseline, follow these steps:

1. Create a repository and a cookbook inside it:

   ```shell
   chef generate repo chef-repo
   cd chef-repo
   chef generate cookbook cookbooks/onprem_baseline
   ```

   The cookbook contains a `recipes/default.rb` for your recipe, a `metadata.rb`, and a `Policyfile.rb`.

1. Create `cookbooks/onprem_baseline/attributes/default.rb`:

   ```ruby
   # NTP configuration
   default['onprem_baseline']['ntp_servers'] = [
     '0.pool.ntp.org',
     '1.pool.ntp.org',
     '2.pool.ntp.org',
     '3.pool.ntp.org',
   ]

   # SSH security settings
   default['onprem_baseline']['ssh_config'] = {
     'PermitRootLogin' => 'no',
     'PasswordAuthentication' => 'no',
     'MaxAuthTries' => '3',
     'LogLevel' => 'VERBOSE',
   }

   # Kernel parameters for stability
   default['onprem_baseline']['kernel_params'] = {
     'net.ipv4.tcp_syncookies' => '1',
     'net.ipv4.tcp_max_syn_backlog' => '2048',
     'net.core.somaxconn' => '1024',
   }
   ```

   Attributes hold values that change between servers or server groups.
   They're centralized in one place so the recipe doesn't have to know about them.

1. Create `cookbooks/onprem_baseline/recipes/default.rb`:

   ```ruby
   # Install and configure NTP for time synchronization
   package 'chrony' do
     action :install
   end

   file '/etc/chrony/chrony.conf.d/onprem-baseline.conf' do
     content lazy {
       lines = node['onprem_baseline']['ntp_servers'].map { |server| "server #{server} iburst" }
       lines.join("\n") + "\n"
     }
     notifies :restart, 'service[chrony]'
   end

   service 'chrony' do
     action [:enable, :start]
   end

   # Manage SSH configuration
   file '/etc/ssh/sshd_config.d/00-onprem-baseline.conf' do
     content lazy {
       node['onprem_baseline']['ssh_config'].map { |key, value| "#{key} #{value}" }.join("\n") + "\n"
     }
     action :create
     notifies :run, 'execute[validate-sshd-config]', :immediately
     notifies :restart, 'service[ssh]'
   end

   # Validate SSH configuration before restarting
   execute 'validate-sshd-config' do
     command 'sshd -t'
     action :nothing
   end

   service 'ssh' do
     action [:enable, :start]
   end

   # Apply kernel parameters for stability
   node['onprem_baseline']['kernel_params'].each do |param, value|
     sysctl param do
       value value
       action :apply
     end
   end
   ```

   The recipe uses built-in resources (`package`, `file`, `service`, `sysctl`, `execute`) to describe the desired state.
   Chef Infra Client compares the current state against this code and makes only the changes needed.

1. Update `cookbooks/onprem_baseline/metadata.rb`:

   ```ruby
   name 'onprem_baseline'
   description 'Manage on-premises server baseline configuration'
   license 'Apache-2.0'
   version '0.1.0'

   supports 'ubuntu', '>= 22.04'
   supports 'redhat', '>= 8.0'
   ```

   The `supports` lines declare which platforms this cookbook targets.

## Bootstrap servers to Chef

Bootstrapping registers a server with Chef and installs Chef Infra Client.
Once bootstrapped, the server can download cookbooks and policies and converge itself on a schedule.

From your Chef Workstation, bootstrap each server in your fleet:

```bash
knife bootstrap <SERVER_IP_ADDRESS> \
  --chef-license accept-silent \
  --run-list 'recipe[onprem_baseline]' \
  --policy-name onprem-prod \
  --policy-group production \
  --user ubuntu \
  --sudo
```

### What this command does

1. Connects to the server via SSH
2. Downloads and installs Chef Infra Client
3. Registers the server with Chef 360 Platform DSM (or Chef Infra Server)
4. Applies the policy named `onprem-prod` and assigns it to the `production` group
5. Runs Chef Infra Client once to converge the server
6. Schedules Chef Infra Client to run every 30 minutes

### Bootstrap output

```text
Connecting to SERVER_IP_ADDRESS
...
Installing Chef Omnibus package...
Starting the first Chef Infra Client run...

[2024-11-16T10:30:00+00:00] INFO: Chef Infra Client, version 19.4.46
[2024-11-16T10:30:02+00:00] INFO: Running handlers complete
Chef Infra Client Infra Phase complete, 5/5 resources updated in 15.23s
```

After bootstrap, your servers are ready.

## Run Chef Infra Client and converge servers

Chef Infra Client runs on a schedule (every 30 minutes by default).
During each run, it fetches the latest policies and cookbooks from Chef 360 Platform, compares the actual server state against the desired state, and updates any resources that don't match.

To trigger a run immediately on a server:

```bash
ssh ubuntu@<SERVER_IP_ADDRESS> sudo chef-client
```

### Output from a converged server

All settings already match:

```text
[2024-11-16T10:35:00+00:00] INFO: Chef Infra Client, version 19.4.46
[2024-11-16T10:35:00+00:00] INFO: Running handlers complete
Chef Infra Client Infra Phase complete, 0/5 resources updated in 2.14s
```

When nothing changed (0/5 resources), Chef Infra Client is idempotent: running it again produces no changes.

## Update configuration across your fleet

To update configuration on all servers, change the cookbook and push it to Chef 360 Platform DSM.

For example, to change the NTP pool servers, edit `attributes/default.rb`:

```ruby
default['onprem_baseline']['ntp_servers'] = [
  '0.pool.ntp.org',
  '1.pool.ntp.org',
  'custom-ntp.internal',  # Add your internal NTP server
]
```

Then upload the updated cookbook:

```bash
cd chef-repo
knife cookbook upload cookbooks/onprem_baseline --use-current-user
```

On the next Chef Infra Client run, all servers with the `onprem-prod` policy will download the updated cookbook and converge.
Servers that need to restart services (like chrony) will do so automatically.

For more complex deployments, use [Policyfiles](/client/latest/policy/policyfile/) to version multiple cookbooks together and promote them through environments:

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

Then bootstrap new servers with the same policy name and group.
Existing servers will converge to the new policy on their next run.

## Detect and correct configuration drift

Configuration drift happens when someone makes a manual change.
For example, an administrator might disable NTP temporarily to fix a system issue and forget to re-enable it, or update SSH settings without updating the cookbook.

Chef Infra Client detects and corrects drift automatically.

### Scenario: Drift is introduced

An administrator manually changes the SSH configuration on one server:

```bash
ssh ubuntu@server1.onprem.local sudo sed -i 's/MaxAuthTries 3/MaxAuthTries 5/' /etc/ssh/sshd_config.d/00-onprem-baseline.conf
ssh ubuntu@server1.onprem.local sudo systemctl restart ssh
```

This change doesn't match the cookbook anymore (the cookbook specifies `MaxAuthTries 3`).

### Drift is detected and corrected

On the next scheduled Chef Infra Client run (or when you trigger it manually), drift is corrected:

```bash
ssh ubuntu@server1.onprem.local sudo chef-client
```

### Output

```text
[2024-11-16T11:00:00+00:00] INFO: Chef Infra Client, version 19.4.46
[2024-11-16T11:00:02+00:00] INFO: Processing file[/etc/ssh/sshd_config.d/00-onprem-baseline.conf] action create
[2024-11-16T11:00:02+00:00] INFO: file[/etc/ssh/sshd_config.d/00-onprem-baseline.conf] updated content
[2024-11-16T11:00:02+00:00] INFO: notify_service[ssh] action delayed
[2024-11-16T11:00:02+00:00] INFO: service[ssh] restarted
[2024-11-16T11:00:04+00:00] INFO: Running handlers complete
Chef Infra Client Infra Phase complete, 2/5 resources updated in 4.32s
```

The file was reverted to match the cookbook, and the SSH service was restarted.
The drift is corrected, and the server is back in the known state defined by the code.

### Multi-server visibility

When you have dozens or hundreds of servers, you need visibility into which ones have drifted.
Chef 360 Platform DSM and [Chef Automate](/automate/client_runs/) give you dashboards showing:

- Which servers have drifted
- What resources failed to converge
- Detailed logs of changes on each server
- Node status and last successful run time

## Manage environment-specific configuration

Different server groups often need different settings.
Production servers might have more NTP servers than development ones.
Different environments might have different SSH key algorithms or logging levels.

Use [attributes](/client/latest/cookbooks/attributes/) and [environments](/client/latest/policy/environments/) to manage these differences without duplicating cookbooks.

### Define environment attributes in Chef 360 Platform DSM

```ruby
# Production environment
{
  "onprem_baseline" => {
    "ntp_servers" => ["0.pool.ntp.org", "1.pool.ntp.org", "2.pool.ntp.org", "3.pool.ntp.org"],
    "ssh_config" => { "LogLevel" => "VERBOSE" },
  }
}

# Development environment
{
  "onprem_baseline" => {
    "ntp_servers" => ["0.pool.ntp.org", "1.pool.ntp.org"],
    "ssh_config" => { "LogLevel" => "DEBUG" },
  }
}
```

Then assign servers to environments during bootstrap:

```bash
knife bootstrap <DEV_SERVER> --policy-name onprem-dev --policy-group development
knife bootstrap <PROD_SERVER> --policy-name onprem-prod --policy-group production
```

Each group receives its own attribute values automatically.

## Integrate compliance

Configuration management and compliance checking work together.
Once your servers are converged to a known state, you can audit them to verify they meet security and regulatory requirements.

Chef Infra Client supports the [Compliance Phase](/client/latest/features/chef_compliance_phase/), which runs [Chef InSpec](/inspec/latest/) audits after converging.

Enable the Compliance Phase in your policy:

```ruby
# chef-repo/Policyfile.rb
name 'onprem-prod'
run_list 'recipe[onprem_baseline]'

audit_cookbook_version '7.9.0'
run_list_audit 'recipe[audit::default]'
```

Create a Chef InSpec profile that audits your on-premises baseline:

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

Chef Automate provides dashboards and reports that show:

- Run status for each node (converged, failed, unreachable)
- Resources that changed in each run
- Nodes that are out of compliance
- Node attribute values and run history
- Customizable reports for auditors and managers

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
- **Bootstrap nodes:** See [Bootstrap servers](/client/latest/install/bootstrap/) for detailed bootstrapping options and troubleshooting
- **Monitor compliance:** Set up [Chef InSpec audits](/inspec/latest/) and [Compliance Phase](/client/latest/features/chef_compliance_phase/) integration
- **Review Chef Automate:** Use [Chef Automate](/automate/client_runs/) for fleet-wide visibility and reporting
- **Understand Chef 360 Platform:** Explore [Chef 360 Platform DSM](/360/latest/understand/dsm/) for centralized policy and cookbook management
- **Migrate from Chef Infra Server:** If you currently use [Chef Infra Server](/server/), see [Migrate to Chef 360 Platform](/360/latest/migrate/on_prem_to_self_managed/)
