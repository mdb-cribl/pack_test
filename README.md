# Cribl Microsoft Entra ID Event Hub IO
----

## About this Pack
This pack is built as a complete SOURCE + DESTINATION solution (identified by the IO suffix). Data collection and delivery happen entirely within the Pack's context - you can choose how data arrives at a DESTINATION:
*  *Send to Worker Group Routes* (the default): data is sent to the top-level Worker Group Routes.
*  *Default Destination*: data is sent to the Worker Group's [Default Destination](https://docs.cribl.io/stream/destinations-default/). 
*  *In-Pack Destination*: data is sent to one or more Destinations configured within the Pack.

This Pack is designed to collect, process, and output Microsoft Entra Activity logs via Azure EventHub. It supports all data that can be configured via [Microsoft Entra diagnostic settings](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-configure-diagnostic-settings) but has specific data reduction support for:
* SigninLogs
* NonInteractiveSigninLogs
* MicrosoftGraphActivityLogs

Splunk data is mapped to the `azure:monitor:aad` sourcetype - this is the sourcetype used by the [Splunk Add-on for Microsoft Cloud Services](https://splunkbase.splunk.com/app/3110) when configuring an [EventHub input](https://splunk.github.io/splunk-add-on-for-microsoft-cloud-services/Sourcetypes/) for ingesting Authentication data.

## Deployment

* Every bundled Source within this pack adds a hidden field: `__packsource`. This field allows for simplified routing based on the Pack source.
* This pack is configured by default to use the Destination *Send to Worker Group Routes*. You *must* add either a Worker Group Route or rely on the Default Destination.
* To explicitly use the Worker Group's *Default Destination*, change the Pack's Routes to *default:default*. The Pack will then route the data to the destination currently set as the Default on the Worker Group.


### Configure Activity Logging into an Azure Event Hub

Follow these general directions to get Entra Activity data streaming into an Event Hub:
* [Register a new Entra Application](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app). You can also use an existing Application if you also have the Client Secret or can have one created. 
* Create an [Azure EventHub](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-create). You can use an existing EventHub Namespace but you *must* use a separate Event Hub for the Entra Activity logging.
* Grant your Entra Application [access to the Event Hub](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal). It needs at least the `Azure Event Hubs Data Sender` and `Azure Event Hubs Data Receiver` roles. 
* [Configure Activity Log diagnostic settings](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-configure-diagnostic-settings). Consider enabling the following Activity Logs at a minimum:
  * AuditLogs
  * SignInLogs
  * NonInteractiveUserSignInLogs
  * ServicePrincipalSignInLogs
  * ManagedIdentitySignInLogs
  * ProvisioningLogs
  * RiskyUsers
  * UserRiskEvents
  * RiskyServicePrincipals
  * ServicePrincipalRiskEvents
  * MicrosoftGraphActivityLogs
  * MicrosoftServicePrincipalSignInLogs

### Obtain Event Hub Authentication Information

There are [two methods](https://docs.cribl.io/stream/sources-azure-event-hubs/#authentication-settings) to authenticate an Event Hub Source in Cribl: OATH and PLAIN. 
* OAUTH - Use this method if you have an Entra Application ID and Secret.
  *  Obtain your Tenant ID and Client ID. They are available from the Application Overview screen.
  *  Obtain the Client secret - if this is unknown then have an Entra Admin create an additional secret for the Application. 
*  PLAIN - Use this method if you do not have a Client ID or if your Entra Admin prefers it. See [this Cribl Blog](https://cribl.io/blog/azure-event-hubs-in-cribl-stream/#Setup) entry for details. 

### Configure the Pack Event Hub Source

To configure the Pack's included Event Hub Source:
* Under `General Settings`, configure the following:
  * Broker - this will likely be in the form `YOUR_BROKER.servicebus.windows.net`.
  * Name of your Event Hub.
* Under Authentication:
  * For OATH, enter your Tenant ID, Client ID, and Client Secret, and Scope. The Scope will be in the format of `https://NAMESPACE.servicebus.windows.net/.default`.
  * For PLAIN, enter the Event Hub's connection string’s primary or secondary key.


See the [Cribl documentation](https://docs.cribl.io/stream/sources-azure-event-hubs/) for complete instructions on configuring an Event Hub Source. 

### Reduction and Sampling Configuration

The Pack contains two optional methods for reducing data volume while still retaining data validity.

#### Conditional Access Policy Filtering
Interactive/Non-Interactive Signin log events contain a large array of all possible Conditional Access Policies that *could* have been applied. Typically, only a few entries contain something useful e.g. they were applied either successfully or unsuccessfully. The remaining are simply labeled "notApplied". By default (controlled by a Variable - see below), this Pack will filter out all non-applied entries via a Code Function in the `cribl_eventhub_msentra_signin_reduction` pipeline. See the pipeline comments for details on how to tune the filtering. 

#### MS Graph Activity Data
This Graph API logs tend to be very large with a very high percentage of `responseStatusCode=200` entries. The `cribl_eventhub_graph_activity_reduction` pipeline contains two configurable Functions to reduce volume via sampling:
* `responseStatusCode=200` events are sampled at a rate of 250 - adjust based on your volume of activity.
* Non-200 status's are [Dynamically Sampled](https://docs.cribl.io/stream/dynamic-sampling-function/) using a combination of User Agent and Status. Adjust the Sample Mode and/or Sample Group Key based on your activity.

### Commit and Deploy
Once everything is configured, perform a Commit & Deploy to enable data collection.

### Variables

The Pack has the following variables:
* `enable_filter_conditional_access_policies`: Controls whether to filter low-value Conditional Access policy array entries. Defaults to `true`.
* `enable_graph_activity_sampling`: Controls whether to sample MS Graph Activity data. Defaults to `true`.
* `enable_move_properties_array`: EXPERIMENTAL: Should the "properties" array contents be moved into the main event body? Use only for Cribl Search.
* `msentra_default_splunk_index`: Default index for the Splunk output - defaults to `msentra`.


## Upgrades

Upgrading certain Cribl Packs using the same Pack ID can have unintended consequences. See [Upgrading an Existing Pack](https://docs.cribl.io/stream/packs#upgrading) for details.

### Version 1.0.0
- Initial release

## Contributing to the Pack

To contribute to the Pack, please connect with us on [Cribl Community Slack](https://cribl-community.slack.com/). You can suggest new features or offer to collaborate.

## License
This Pack uses the following license: [Apache 2.0](https://github.com/criblio/appscope/blob/master/LICENSE).
