# Azure vWAN S2S VPN Monitoring with Zabbix 7.4

## Purpose

This document describes the production design and configuration for monitoring Azure Virtual WAN site-to-site VPN tunnels and BGP sessions in Zabbix 7.4.16.

The solution uses:

- Azure Virtual WAN VPN Gateway platform metrics
- Microsoft Entra ID application authentication with client credentials
- A single Zabbix Script master item polling Azure every five minutes
- Dependent low-level discovery rules for VPN tunnels and correlated S2S BGP peers
- Dependent item prototypes for traffic, packet drops, BGP status, and route counts
- Azure VPN-site configuration to correlate gateway-level BGP metrics with the correct VPN site

The implementation avoids relying on Check Point `vpnt` interface SNMP counters, which may remain at zero even when traffic is flowing.

---

## Environment used for this implementation

| Component | Value |
|---|---|
| Zabbix | 7.4.16 |
| Azure subscription | `ac4bd05a-4a33-40cb-837d-12761022c06e` |
| Resource group | `weu-vwan` |
| Virtual hub | `WEU_HUB1` |
| vWAN VPN Gateway | `0d58457476444e4a91237f41012f10cd-westeurope-gw` |
| VPN Gateway resource ID | `/subscriptions/ac4bd05a-4a33-40cb-837d-12761022c06e/resourceGroups/weu-vwan/providers/Microsoft.Network/vpnGateways/0d58457476444e4a91237f41012f10cd-westeurope-gw` |
| Azure vWAN ASN | `65515` |
| Gateway scale unit | `2` |
| Check Point release | `R82` |
| Example VPN site | `gw-hq-dk` |
| Example remote public IP | `195.93.192.4` |
| Check Point cluster BGP VIP | `169.254.230.4` |
| Check Point cluster BGP ASN | `65230` |
| Active member BGP address | `169.254.230.5` |
| Standby member BGP address | `169.254.230.6` |
| Azure Instance0 BGP address | `10.248.0.12` |
| Azure Instance1 BGP address | `10.248.0.13` |

### Confirmed HQ BGP topology

```text
Check Point ClusterXL
  Cluster BGP VIP: 169.254.230.4, AS 65230
  Member 1:         169.254.230.5
  Member 2:         169.254.230.6

  169.254.230.4 <---- BGP ----> 10.248.0.12, Azure Instance0
  169.254.230.4 <---- BGP ----> 10.248.0.13, Azure Instance1
```

Azure monitors the stable cluster VIP `169.254.230.4`. The physical ClusterXL member addresses `.5` and `.6` must not be hard-coded into Azure-side discovery.

---

## Monitoring flow

```text
Microsoft Entra ID
  Token endpoint
       |
       v
Zabbix Script master item, every 5 minutes
       |
       +-- Read Azure VPN Gateway configuration
       |     +-- Connected remote VPN-site IDs
       |     +-- Instance0/Instance1 local BGP addresses
       |
       +-- List VPN sites in the gateway resource group
       |     +-- Site and link names
       |     +-- Remote public IP
       |     +-- BGP peer address and ASN
       |     +-- Provider and link speed
       |
       +-- Query Azure Monitor tunnel metrics
       |     +-- TunnelIngressBytes
       |     +-- TunnelEgressBytes
       |     +-- TunnelIngressPacketDropCount
       |     +-- TunnelEgressPacketDropCount
       |
       +-- Query Azure Monitor BGP metrics
       |     +-- BgpPeerStatus
       |     +-- BgpRoutesLearned
       |     +-- BgpRoutesAdvertised
       |
       +-- Normalize and correlate JSON
             |
             +-- Tunnel LLD
             |     +-- Traffic items
             |     +-- Packet-drop items
             |     +-- Tunnel graphs
             |
             +-- Correlated S2S BGP LLD
                   +-- BGP status
                   +-- Route counts
                   +-- BGP triggers and graphs
```

---

## Requirements

### Azure

1. Microsoft Entra ID application registration.
2. A valid client secret.
3. At least `Reader` access at the subscription, resource group, VPN Gateway, or otherwise sufficient scope covering:
   - `Microsoft.Network/vpnGateways/read`
   - `Microsoft.Network/vpnSites/read`
   - Azure Monitor metrics read access
4. Outbound HTTPS connectivity from the Zabbix server or assigned Zabbix proxy to:
   - `https://login.microsoftonline.com`
   - `https://management.azure.com`
5. DNS resolution and TCP/443 connectivity to those endpoints.

### Zabbix

1. Zabbix 7.4.16.
2. A logical host such as `Azure vWAN WEU VPN Gateway`.
3. No Zabbix agent interface is required for the Script master item.
4. The Zabbix server or selected proxy must execute Script items and reach Azure.
5. Recommended timeout for the master item: `30s`.

### Azure metric intervals

The design polls every five minutes and uses `PT5M` metric buckets. Each `TunnelIngressBytes` or `TunnelEgressBytes` sample is the number of bytes in that five-minute bucket, not a cumulative interface counter.

Average bits per second are calculated as:

```text
bps = bytes * 8 / 300
```

---

## Zabbix host macros

Create these macros under:

`Data collection -> Hosts -> Azure vWAN WEU VPN Gateway -> Macros`

| Macro | Type | Value |
|---|---|---|
| `{$AZURE.TENANT.ID}` | Text | Microsoft Entra tenant ID |
| `{$AZURE.APP.ID}` | Text | Application/client ID |
| `{$AZURE.APP.SECRET}` | Secret text | Application client secret |
| `{$AZURE.VWAN.RESOURCE.ID}` | Text | Full VPN Gateway resource ID |

Use Secret text for `{$AZURE.APP.SECRET}`. Do not paste a secret directly into JavaScript.

---

# Master item

Create or edit the item under:

`Data collection -> Hosts -> Azure vWAN WEU VPN Gateway -> Items`

| Field | Value |
|---|---|
| Name | `Azure vWAN VPN: Get all tunnel metrics` |
| Type | Script |
| Key | `azure.vwan.vpn.all.get` |
| Type of information | Text |
| Update interval | `5m` |
| History | `1d` |
| Trends | Do not keep |
| Timeout | `30s` |

## Master item parameters

| Name | Value |
|---|---|
| `tenant_id` | `{$AZURE.TENANT.ID}` |
| `client_id` | `{$AZURE.APP.ID}` |
| `client_secret` | `{$AZURE.APP.SECRET}` |
| `resource_id` | `{$AZURE.VWAN.RESOURCE.ID}` |

Remove obsolete per-site parameters such as `connection_name` and `remote_ip` from this all-tunnel item.

## Full master JavaScript

```javascript
var p = JSON.parse(value);

function fail(message, body) {
    if (body !== undefined && body !== null && body !== "") {
        throw message + ": " + body;
    }
    throw message;
}

function encodeForm(input) {
    return encodeURIComponent(input).replace(/%20/g, "+");
}

function dimensionValue(metadata, wantedName) {
    var i;
    var actualName;

    if (!metadata) {
        return null;
    }

    for (i = 0; i < metadata.length; i++) {
        if (!metadata[i].name || metadata[i].name.value === undefined) {
            continue;
        }

        actualName = String(metadata[i].name.value).toLowerCase();
        if (actualName === wantedName.toLowerCase()) {
            return String(metadata[i].value);
        }
    }

    return null;
}

function latestSample(data, fieldName) {
    var i;

    if (!data || data.length === 0) {
        return null;
    }

    for (i = data.length - 1; i >= 0; i--) {
        if (data[i][fieldName] !== null && data[i][fieldName] !== undefined) {
            return {
                timestamp: data[i].timeStamp,
                value: Number(data[i][fieldName])
            };
        }
    }

    return null;
}

function azureGet(url, accessToken, description) {
    var request = new HttpRequest();
    var response;
    var status;
    var result;

    request.addHeader("Authorization: Bearer " + accessToken);
    request.addHeader("Accept: application/json");

    response = request.get(url);
    status = request.getStatus();

    if (status < 200 || status >= 300) {
        fail(description + " failed with HTTP " + status, response);
    }

    try {
        result = JSON.parse(response);
    }
    catch (error) {
        fail(description + " returned invalid JSON", response);
    }

    return result;
}

function getPagedAzureCollection(firstUrl, accessToken, description) {
    var items = [];
    var currentUrl = firstUrl;
    var page;
    var i;
    var pageCount = 0;

    while (currentUrl) {
        pageCount++;

        if (pageCount > 20) {
            fail(description + " exceeded the pagination safety limit");
        }

        page = azureGet(currentUrl, accessToken, description);

        if (!page.value || !(page.value instanceof Array)) {
            fail(description + " returned no value array");
        }

        for (i = 0; i < page.value.length; i++) {
            items.push(page.value[i]);
        }

        currentUrl = page.nextLink || null;
    }

    return items;
}

function parseGatewayResourceId(resourceId) {
    var match = resourceId.match(
        /^\/subscriptions\/([^\/]+)\/resourceGroups\/([^\/]+)\/providers\/Microsoft\.Network\/vpnGateways\/([^\/]+)$/i
    );

    if (!match) {
        fail("resource_id is not a valid Microsoft.Network/vpnGateways resource ID");
    }

    return {
        subscription_id: match[1],
        resource_group: match[2],
        gateway_name: match[3]
    };
}

function tunnelKey(connection, instance, remoteIp) {
    return connection + "|" + instance + "|" + remoteIp;
}

function bgpKey(siteName, peerAddress, instance) {
    return siteName + "|" + peerAddress + "|" + instance;
}

function compareTunnels(a, b) {
    if (a.connection < b.connection) return -1;
    if (a.connection > b.connection) return 1;
    if (a.instance < b.instance) return -1;
    if (a.instance > b.instance) return 1;
    if (a.remote_ip < b.remote_ip) return -1;
    if (a.remote_ip > b.remote_ip) return 1;
    return 0;
}

function compareBgpPeers(a, b) {
    if (a.site_name < b.site_name) return -1;
    if (a.site_name > b.site_name) return 1;
    if (a.instance < b.instance) return -1;
    if (a.instance > b.instance) return 1;
    if (a.peer_address < b.peer_address) return -1;
    if (a.peer_address > b.peer_address) return 1;
    return 0;
}

if (!p.tenant_id) fail("Missing tenant_id parameter");
if (!p.client_id) fail("Missing client_id parameter");
if (!p.client_secret) fail("Missing client_secret parameter");
if (!p.resource_id) fail("Missing resource_id parameter");

var gatewayIdentity = parseGatewayResourceId(p.resource_id);

var tokenRequest = new HttpRequest();
tokenRequest.addHeader("Content-Type: application/x-www-form-urlencoded");
tokenRequest.addHeader("Accept: application/json");

var tokenUrl =
    "https://login.microsoftonline.com/" +
    encodeURIComponent(p.tenant_id) +
    "/oauth2/v2.0/token";

var tokenBody =
    "client_id=" + encodeForm(p.client_id) +
    "&client_secret=" + encodeForm(p.client_secret) +
    "&scope=" + encodeForm("https://management.azure.com/.default") +
    "&grant_type=client_credentials";

var tokenResponse = tokenRequest.post(tokenUrl, tokenBody);
var tokenStatus = tokenRequest.getStatus();

if (tokenStatus < 200 || tokenStatus >= 300) {
    fail("Azure token request failed with HTTP " + tokenStatus, tokenResponse);
}

var tokenJson;
try {
    tokenJson = JSON.parse(tokenResponse);
}
catch (error) {
    fail("Azure token response is not valid JSON", tokenResponse);
}

if (!tokenJson.access_token) {
    fail("Azure token response contains no access_token", tokenResponse);
}

var accessToken = tokenJson.access_token;

var gatewayUrl =
    "https://management.azure.com" +
    p.resource_id +
    "?api-version=2025-01-01";

var gatewayJson = azureGet(
    gatewayUrl,
    accessToken,
    "Azure VPN Gateway configuration request"
);

var connectedSiteIds = {};
var connectionNameBySiteId = {};
var gatewayConnections =
    gatewayJson.properties && gatewayJson.properties.connections
        ? gatewayJson.properties.connections
        : [];

var i;
var connection;
var remoteSiteId;

for (i = 0; i < gatewayConnections.length; i++) {
    connection = gatewayConnections[i];

    if (!connection.properties ||
            !connection.properties.remoteVpnSite ||
            !connection.properties.remoteVpnSite.id) {
        continue;
    }

    remoteSiteId = String(connection.properties.remoteVpnSite.id).toLowerCase();
    connectedSiteIds[remoteSiteId] = true;
    connectionNameBySiteId[remoteSiteId] = connection.name || null;
}

var azureLocalBgpByInstance = {};
var bgpAddressConfigs =
    gatewayJson.properties &&
    gatewayJson.properties.bgpSettings &&
    gatewayJson.properties.bgpSettings.bgpPeeringAddresses
        ? gatewayJson.properties.bgpSettings.bgpPeeringAddresses
        : [];

var bgpConfig;
var localAddresses;

for (i = 0; i < bgpAddressConfigs.length; i++) {
    bgpConfig = bgpAddressConfigs[i];

    if (!bgpConfig.ipconfigurationId) {
        continue;
    }

    localAddresses = bgpConfig.defaultBgpIpAddresses || [];

    if (localAddresses.length > 0) {
        azureLocalBgpByInstance[bgpConfig.ipconfigurationId] =
            String(localAddresses[0]);
    }
}

var vpnSitesUrl =
    "https://management.azure.com/subscriptions/" +
    encodeURIComponent(gatewayIdentity.subscription_id) +
    "/resourceGroups/" +
    encodeURIComponent(gatewayIdentity.resource_group) +
    "/providers/Microsoft.Network/vpnSites" +
    "?api-version=2025-01-01";

var allVpnSites = getPagedAzureCollection(
    vpnSitesUrl,
    accessToken,
    "Azure VPN site list request"
);

var siteLinks = [];
var siteByBgpPeer = {};
var site;
var siteId;
var links;
var link;
var linkProperties;
var linkBgp;
var siteInfo;
var j;

for (i = 0; i < allVpnSites.length; i++) {
    site = allVpnSites[i];

    if (!site.id) {
        continue;
    }

    siteId = String(site.id).toLowerCase();

    if (!connectedSiteIds[siteId]) {
        continue;
    }

    links = site.properties && site.properties.vpnSiteLinks
        ? site.properties.vpnSiteLinks
        : [];

    for (j = 0; j < links.length; j++) {
        link = links[j];
        linkProperties = link.properties || {};
        linkBgp = linkProperties.bgpProperties || {};

        if (!linkBgp.bgpPeeringAddress) {
            continue;
        }

        siteInfo = {
            site_id: site.id,
            site_name: site.name,
            connection_resource_name: connectionNameBySiteId[siteId],
            link_name: link.name,
            remote_public_ip: linkProperties.ipAddress || null,
            bgp_peer_address: String(linkBgp.bgpPeeringAddress),
            bgp_asn: linkBgp.asn !== undefined ? Number(linkBgp.asn) : null,
            link_provider:
                linkProperties.linkProperties &&
                linkProperties.linkProperties.linkProviderName
                    ? linkProperties.linkProperties.linkProviderName
                    : null,
            link_speed_mbps:
                linkProperties.linkProperties &&
                linkProperties.linkProperties.linkSpeedInMbps !== undefined
                    ? Number(linkProperties.linkProperties.linkSpeedInMbps)
                    : null
        };

        siteLinks.push(siteInfo);

        if (!siteByBgpPeer[siteInfo.bgp_peer_address]) {
            siteByBgpPeer[siteInfo.bgp_peer_address] = [];
        }

        siteByBgpPeer[siteInfo.bgp_peer_address].push(siteInfo);
    }
}

var metricsBaseUrl =
    "https://management.azure.com" +
    p.resource_id +
    "/providers/microsoft.insights/metrics";

var tunnelFilter =
    "ConnectionName eq '*' and " +
    "RemoteIP eq '*' and " +
    "Instance eq '*'";

var tunnelMetricNames =
    "TunnelIngressBytes," +
    "TunnelEgressBytes," +
    "TunnelIngressPacketDropCount," +
    "TunnelEgressPacketDropCount";

var tunnelMetricsUrl =
    metricsBaseUrl +
    "?api-version=2023-10-01" +
    "&metricnamespace=" + encodeURIComponent("Microsoft.Network/vpnGateways") +
    "&metricnames=" + encodeURIComponent(tunnelMetricNames) +
    "&timespan=" + encodeURIComponent("PT6H") +
    "&aggregation=" + encodeURIComponent("Total") +
    "&interval=" + encodeURIComponent("PT5M") +
    "&$filter=" + encodeURIComponent(tunnelFilter);

var tunnelMetricsJson = azureGet(
    tunnelMetricsUrl,
    accessToken,
    "Azure tunnel metrics request"
);

var bgpFilter =
    "BgpPeerAddress eq '*' and " +
    "Instance eq '*'";

var bgpStatusUrl =
    metricsBaseUrl +
    "?api-version=2023-10-01" +
    "&metricnamespace=" + encodeURIComponent("Microsoft.Network/vpnGateways") +
    "&metricnames=" + encodeURIComponent("BgpPeerStatus") +
    "&timespan=" + encodeURIComponent("PT6H") +
    "&aggregation=" + encodeURIComponent("Average") +
    "&interval=" + encodeURIComponent("PT5M") +
    "&$filter=" + encodeURIComponent(bgpFilter);

var bgpStatusJson = azureGet(
    bgpStatusUrl,
    accessToken,
    "Azure BGP status request"
);

var bgpRoutesUrl =
    metricsBaseUrl +
    "?api-version=2023-10-01" +
    "&metricnamespace=" + encodeURIComponent("Microsoft.Network/vpnGateways") +
    "&metricnames=" + encodeURIComponent("BgpRoutesLearned,BgpRoutesAdvertised") +
    "&timespan=" + encodeURIComponent("PT6H") +
    "&aggregation=" + encodeURIComponent("Total") +
    "&interval=" + encodeURIComponent("PT5M") +
    "&$filter=" + encodeURIComponent(bgpFilter);

var bgpRoutesJson = azureGet(
    bgpRoutesUrl,
    accessToken,
    "Azure BGP route metrics request"
);

var tunnelMap = {};
var metricIndex;
var seriesIndex;
var metric;
var metricName;
var metricRole;
var series;
var metricConnection;
var instance;
var remoteIp;
var key;
var sample;

if (!tunnelMetricsJson.value || !(tunnelMetricsJson.value instanceof Array)) {
    fail("Azure tunnel metrics response has no value array");
}

for (metricIndex = 0; metricIndex < tunnelMetricsJson.value.length; metricIndex++) {
    metric = tunnelMetricsJson.value[metricIndex];

    if (!metric.name || metric.name.value === undefined) {
        continue;
    }

    metricName = metric.name.value;

    if (metricName === "TunnelIngressBytes") {
        metricRole = "ingress";
    }
    else if (metricName === "TunnelEgressBytes") {
        metricRole = "egress";
    }
    else if (metricName === "TunnelIngressPacketDropCount") {
        metricRole = "ingress_drops";
    }
    else if (metricName === "TunnelEgressPacketDropCount") {
        metricRole = "egress_drops";
    }
    else {
        continue;
    }

    if (!metric.timeseries) {
        continue;
    }

    for (seriesIndex = 0; seriesIndex < metric.timeseries.length; seriesIndex++) {
        series = metric.timeseries[seriesIndex];

        metricConnection = dimensionValue(series.metadatavalues, "ConnectionName");
        instance = dimensionValue(series.metadatavalues, "Instance");
        remoteIp = dimensionValue(series.metadatavalues, "RemoteIP");

        if (!metricConnection || !instance || !remoteIp) {
            continue;
        }

        key = tunnelKey(metricConnection, instance, remoteIp);

        if (!tunnelMap[key]) {
            tunnelMap[key] = {
                tunnel_key: key,
                connection: metricConnection,
                instance: instance,
                remote_ip: remoteIp,
                azure_local_bgp_ip: azureLocalBgpByInstance[instance] || null,
                ingress_bytes: null,
                ingress_bps: null,
                ingress_timestamp: null,
                egress_bytes: null,
                egress_bps: null,
                egress_timestamp: null,
                ingress_packet_drops: null,
                ingress_drops_timestamp: null,
                egress_packet_drops: null,
                egress_drops_timestamp: null
            };
        }

        sample = latestSample(series.data, "total");

        if (sample === null) {
            continue;
        }

        if (metricRole === "ingress") {
            tunnelMap[key].ingress_bytes = sample.value;
            tunnelMap[key].ingress_bps = sample.value * 8 / 300;
            tunnelMap[key].ingress_timestamp = sample.timestamp;
        }
        else if (metricRole === "egress") {
            tunnelMap[key].egress_bytes = sample.value;
            tunnelMap[key].egress_bps = sample.value * 8 / 300;
            tunnelMap[key].egress_timestamp = sample.timestamp;
        }
        else if (metricRole === "ingress_drops") {
            tunnelMap[key].ingress_packet_drops = sample.value;
            tunnelMap[key].ingress_drops_timestamp = sample.timestamp;
        }
        else if (metricRole === "egress_drops") {
            tunnelMap[key].egress_packet_drops = sample.value;
            tunnelMap[key].egress_drops_timestamp = sample.timestamp;
        }
    }
}

var rawBgpMap = {};
var bgpGroups = [
    { json: bgpStatusJson, sample_field: "average" },
    { json: bgpRoutesJson, sample_field: "total" }
];

var groupIndex;
var group;
var peerAddress;
var rawBgpKey;

for (groupIndex = 0; groupIndex < bgpGroups.length; groupIndex++) {
    group = bgpGroups[groupIndex];

    if (!group.json.value || !(group.json.value instanceof Array)) {
        continue;
    }

    for (metricIndex = 0; metricIndex < group.json.value.length; metricIndex++) {
        metric = group.json.value[metricIndex];

        if (!metric.name || metric.name.value === undefined || !metric.timeseries) {
            continue;
        }

        metricName = metric.name.value;

        for (seriesIndex = 0; seriesIndex < metric.timeseries.length; seriesIndex++) {
            series = metric.timeseries[seriesIndex];

            peerAddress = dimensionValue(series.metadatavalues, "BgpPeerAddress");
            instance = dimensionValue(series.metadatavalues, "Instance");

            if (!peerAddress || !instance) {
                continue;
            }

            rawBgpKey = peerAddress + "|" + instance;

            if (!rawBgpMap[rawBgpKey]) {
                rawBgpMap[rawBgpKey] = {
                    peer_address: peerAddress,
                    instance: instance,
                    status: null,
                    status_timestamp: null,
                    routes_learned: null,
                    routes_learned_timestamp: null,
                    routes_advertised: null,
                    routes_advertised_timestamp: null
                };
            }

            sample = latestSample(series.data, group.sample_field);

            if (sample === null) {
                continue;
            }

            if (metricName === "BgpPeerStatus") {
                rawBgpMap[rawBgpKey].status = sample.value;
                rawBgpMap[rawBgpKey].status_timestamp = sample.timestamp;
            }
            else if (metricName === "BgpRoutesLearned") {
                rawBgpMap[rawBgpKey].routes_learned = sample.value;
                rawBgpMap[rawBgpKey].routes_learned_timestamp = sample.timestamp;
            }
            else if (metricName === "BgpRoutesAdvertised") {
                rawBgpMap[rawBgpKey].routes_advertised = sample.value;
                rawBgpMap[rawBgpKey].routes_advertised_timestamp = sample.timestamp;
            }
        }
    }
}

var bgpPeers = [];
var unmatchedBgpPeers = [];
var rawPeer;
var matchingSites;
var matchingSite;
var bgpObject;
var siteIndex;

for (key in rawBgpMap) {
    if (!rawBgpMap.hasOwnProperty(key)) {
        continue;
    }

    rawPeer = rawBgpMap[key];
    matchingSites = siteByBgpPeer[rawPeer.peer_address];

    if (!matchingSites || matchingSites.length === 0) {
        unmatchedBgpPeers.push(rawPeer);
        continue;
    }

    for (siteIndex = 0; siteIndex < matchingSites.length; siteIndex++) {
        matchingSite = matchingSites[siteIndex];

        bgpObject = {
            bgp_key: bgpKey(
                matchingSite.site_name,
                rawPeer.peer_address,
                rawPeer.instance
            ),
            site_name: matchingSite.site_name,
            connection_resource_name: matchingSite.connection_resource_name,
            link_name: matchingSite.link_name,
            remote_public_ip: matchingSite.remote_public_ip,
            peer_address: rawPeer.peer_address,
            peer_asn: matchingSite.bgp_asn,
            instance: rawPeer.instance,
            azure_local_bgp_ip:
                azureLocalBgpByInstance[rawPeer.instance] || null,
            link_provider: matchingSite.link_provider,
            link_speed_mbps: matchingSite.link_speed_mbps,
            status: rawPeer.status,
            status_timestamp: rawPeer.status_timestamp,
            routes_learned: rawPeer.routes_learned,
            routes_learned_timestamp: rawPeer.routes_learned_timestamp,
            routes_advertised: rawPeer.routes_advertised,
            routes_advertised_timestamp: rawPeer.routes_advertised_timestamp
        };

        bgpPeers.push(bgpObject);
    }
}

var tunnels = [];

for (key in tunnelMap) {
    if (tunnelMap.hasOwnProperty(key)) {
        tunnels.push(tunnelMap[key]);
    }
}

if (tunnels.length === 0) {
    fail("Azure returned no discoverable VPN tunnel metrics");
}

tunnels.sort(compareTunnels);
bgpPeers.sort(compareBgpPeers);

return JSON.stringify({
    collected_at: new Date().toISOString(),
    resource_id: p.resource_id,
    subscription_id: gatewayIdentity.subscription_id,
    resource_group: gatewayIdentity.resource_group,
    gateway_name: gatewayIdentity.gateway_name,
    tunnel_count: tunnels.length,
    connected_site_link_count: siteLinks.length,
    bgp_peer_count: bgpPeers.length,
    unmatched_bgp_peer_count: unmatchedBgpPeers.length,
    tunnels: tunnels,
    vpn_site_links: siteLinks,
    bgp_peers: bgpPeers,
    unmatched_bgp_peers: unmatchedBgpPeers
});
```

---

# Tunnel low-level discovery

Create or edit:

`Data collection -> Hosts -> Azure vWAN WEU VPN Gateway -> Discovery`

| Field | Value |
|---|---|
| Name | `Azure vWAN VPN tunnel discovery` |
| Type | Dependent item |
| Key | `azure.vwan.vpn.tunnel.discovery` |
| Master item | `Azure vWAN VPN: Get all tunnel metrics` |
| Disable lost resources | After `1h` |
| Delete lost resources | After `30d` |

## Tunnel discovery preprocessing

### Step 1: JavaScript

```javascript
var root = JSON.parse(value);
var result = [];
var i;
var tunnel;

if (!root.tunnels || !(root.tunnels instanceof Array)) {
    throw "Master item contains no tunnels array";
}

for (i = 0; i < root.tunnels.length; i++) {
    tunnel = root.tunnels[i];

    if (!tunnel.connection || !tunnel.instance || !tunnel.remote_ip) {
        continue;
    }

    result.push({
        "{#CONNECTION}": String(tunnel.connection),
        "{#INSTANCE}": String(tunnel.instance),
        "{#REMOTEIP}": String(tunnel.remote_ip),
        "{#TUNNELKEY}": String(tunnel.tunnel_key),
        "{#AZUREBGPIP}": tunnel.azure_local_bgp_ip
            ? String(tunnel.azure_local_bgp_ip)
            : ""
    });
}

if (result.length === 0) {
    throw "No valid VPN tunnel objects found";
}

return JSON.stringify(result);
```

### Step 2

```text
Discard unchanged with heartbeat: 1h
```

---

# Tunnel item prototypes

All prototypes below use the master item `Azure vWAN VPN: Get all tunnel metrics`.

## Check Point to Azure traffic

| Field | Value |
|---|---|
| Name | `{#CONNECTION} {#INSTANCE}: Check Point to Azure traffic` |
| Type | Dependent item |
| Key | `azure.vwan.vpn.ingress.bps[{#CONNECTION},{#INSTANCE},{#REMOTEIP}]` |
| Type | Numeric float |
| Units | `bps` |
| History | `30d` |
| Trends | `365d` |

Preprocessing:

```javascript
var root = JSON.parse(value);
var connection = "{#CONNECTION}";
var instance = "{#INSTANCE}";
var remoteIp = "{#REMOTEIP}";
var i;
var tunnel;

if (!root.tunnels || !(root.tunnels instanceof Array)) {
    throw "Master item contains no tunnels array";
}

for (i = 0; i < root.tunnels.length; i++) {
    tunnel = root.tunnels[i];

    if (String(tunnel.connection) === connection &&
            String(tunnel.instance) === instance &&
            String(tunnel.remote_ip) === remoteIp) {
        if (tunnel.ingress_bps === null || tunnel.ingress_bps === undefined) {
            throw "No ingress traffic sample";
        }
        return Number(tunnel.ingress_bps);
    }
}

throw "Requested tunnel was not found";
```

Use `Custom on fail: Discard value`.

## Azure to Check Point traffic

| Field | Value |
|---|---|
| Name | `{#CONNECTION} {#INSTANCE}: Azure to Check Point traffic` |
| Type | Dependent item |
| Key | `azure.vwan.vpn.egress.bps[{#CONNECTION},{#INSTANCE},{#REMOTEIP}]` |
| Type | Numeric float |
| Units | `bps` |
| History | `30d` |
| Trends | `365d` |

Preprocessing:

```javascript
var root = JSON.parse(value);
var connection = "{#CONNECTION}";
var instance = "{#INSTANCE}";
var remoteIp = "{#REMOTEIP}";
var i;
var tunnel;

if (!root.tunnels || !(root.tunnels instanceof Array)) {
    throw "Master item contains no tunnels array";
}

for (i = 0; i < root.tunnels.length; i++) {
    tunnel = root.tunnels[i];

    if (String(tunnel.connection) === connection &&
            String(tunnel.instance) === instance &&
            String(tunnel.remote_ip) === remoteIp) {
        if (tunnel.egress_bps === null || tunnel.egress_bps === undefined) {
            throw "No egress traffic sample";
        }
        return Number(tunnel.egress_bps);
    }
}

throw "Requested tunnel was not found";
```

Use `Custom on fail: Discard value`.

## Ingress packet drops

| Field | Value |
|---|---|
| Name | `{#CONNECTION} {#INSTANCE}: Ingress packet drops` |
| Type | Dependent item |
| Key | `azure.vwan.vpn.ingress.drops[{#CONNECTION},{#INSTANCE},{#REMOTEIP}]` |
| Type | Numeric unsigned |
| Units | `packets` |
| History | `30d` |
| Trends | `365d` |

Preprocessing:

```javascript
var root = JSON.parse(value);
var connection = "{#CONNECTION}";
var instance = "{#INSTANCE}";
var remoteIp = "{#REMOTEIP}";
var i;
var tunnel;

if (!root.tunnels || !(root.tunnels instanceof Array)) {
    throw "Master item contains no tunnels array";
}

for (i = 0; i < root.tunnels.length; i++) {
    tunnel = root.tunnels[i];

    if (String(tunnel.connection) === connection &&
            String(tunnel.instance) === instance &&
            String(tunnel.remote_ip) === remoteIp) {
        if (tunnel.ingress_packet_drops === null ||
                tunnel.ingress_packet_drops === undefined) {
            throw "No ingress packet-drop sample";
        }
        return Number(tunnel.ingress_packet_drops);
    }
}

throw "Requested tunnel was not found";
```

Use `Custom on fail: Discard value`.

## Egress packet drops

| Field | Value |
|---|---|
| Name | `{#CONNECTION} {#INSTANCE}: Egress packet drops` |
| Type | Dependent item |
| Key | `azure.vwan.vpn.egress.drops[{#CONNECTION},{#INSTANCE},{#REMOTEIP}]` |
| Type | Numeric unsigned |
| Units | `packets` |
| History | `30d` |
| Trends | `365d` |

Preprocessing:

```javascript
var root = JSON.parse(value);
var connection = "{#CONNECTION}";
var instance = "{#INSTANCE}";
var remoteIp = "{#REMOTEIP}";
var i;
var tunnel;

if (!root.tunnels || !(root.tunnels instanceof Array)) {
    throw "Master item contains no tunnels array";
}

for (i = 0; i < root.tunnels.length; i++) {
    tunnel = root.tunnels[i];

    if (String(tunnel.connection) === connection &&
            String(tunnel.instance) === instance &&
            String(tunnel.remote_ip) === remoteIp) {
        if (tunnel.egress_packet_drops === null ||
                tunnel.egress_packet_drops === undefined) {
            throw "No egress packet-drop sample";
        }
        return Number(tunnel.egress_packet_drops);
    }
}

throw "Requested tunnel was not found";
```

Use `Custom on fail: Discard value`.

## Recommended tags

| Tag | Value |
|---|---|
| `component` | `vpn` |
| `connection` | `{#CONNECTION}` |
| `instance` | `{#INSTANCE}` |
| `remote-ip` | `{#REMOTEIP}` |

Add `direction=checkpoint-to-azure` or `direction=azure-to-checkpoint` to traffic items, and `metric=packet-drops` to drop items.

---

# Tunnel trigger prototypes

Use the **new packet-drop delta items** documented under Operational notes for packet-drop alerting. Do not trigger on the raw Azure packet-drop counter.

# Tunnel graph prototype

Create under the tunnel discovery rule:

```text
Name: {#CONNECTION} {#INSTANCE}: VPN traffic
Graph type: Normal
Y-axis: Calculated
```

Add:

1. `{#CONNECTION} {#INSTANCE}: Check Point to Azure traffic`
2. `{#CONNECTION} {#INSTANCE}: Azure to Check Point traffic`

Keep both directions positive to preserve straightforward interpretation.

---

# Correlated S2S BGP low-level discovery

Create a second dependent discovery rule:

| Field | Value |
|---|---|
| Name | `Azure vWAN S2S BGP peer discovery` |
| Type | Dependent item |
| Key | `azure.vwan.s2s.bgp.discovery` |
| Master item | `Azure vWAN VPN: Get all tunnel metrics` |
| Disable lost resources | After `1h` |
| Delete lost resources | After `30d` |

## BGP discovery preprocessing

### Step 1: JavaScript

```javascript
var root = JSON.parse(value);
var result = [];
var i;
var peer;

if (!root.bgp_peers || !(root.bgp_peers instanceof Array)) {
    throw "Master item contains no bgp_peers array";
}

for (i = 0; i < root.bgp_peers.length; i++) {
    peer = root.bgp_peers[i];

    if (!peer.site_name || !peer.peer_address ||
            !peer.instance || !peer.bgp_key) {
        continue;
    }

    result.push({
        "{#SITE}": String(peer.site_name),
        "{#LINK}": peer.link_name ? String(peer.link_name) : "",
        "{#BGPPEER}": String(peer.peer_address),
        "{#BGPASN}": peer.peer_asn !== null && peer.peer_asn !== undefined
            ? String(peer.peer_asn)
            : "",
        "{#INSTANCE}": String(peer.instance),
        "{#AZUREBGPIP}": peer.azure_local_bgp_ip
            ? String(peer.azure_local_bgp_ip)
            : "",
        "{#REMOTEIP}": peer.remote_public_ip
            ? String(peer.remote_public_ip)
            : "",
        "{#PROVIDER}": peer.link_provider
            ? String(peer.link_provider)
            : "",
        "{#LINKSPEED}": peer.link_speed_mbps !== null &&
            peer.link_speed_mbps !== undefined
                ? String(peer.link_speed_mbps)
                : "",
        "{#BGPKEY}": String(peer.bgp_key)
    });
}

if (result.length === 0) {
    throw "No connected S2S VPN BGP peers found";
}

return JSON.stringify(result);
```

### Step 2

```text
Discard unchanged with heartbeat: 1h
```

This discovery returns only Azure BGP metric series whose peer address matches the configured BGP peering address of a VPN site connected to this gateway. Unrelated gateway, hub, or NVA metric series remain in `unmatched_bgp_peers` for diagnostics and are not turned into S2S BGP items.

---

# BGP item prototypes

## BGP status

| Field | Value |
|---|---|
| Name | `{#SITE} {#INSTANCE}: BGP status, {#BGPPEER} to {#AZUREBGPIP}` |
| Type | Dependent item |
| Key | `azure.vwan.s2s.bgp.status[{#SITE},{#BGPPEER},{#INSTANCE}]` |
| Master item | `Azure vWAN VPN: Get all tunnel metrics` |
| Type | Numeric unsigned |
| History | `30d` |
| Trends | `365d` |

Preprocessing:

```javascript
var root = JSON.parse(value);
var site = "{#SITE}";
var peerAddress = "{#BGPPEER}";
var instance = "{#INSTANCE}";
var i;
var peer;

if (!root.bgp_peers || !(root.bgp_peers instanceof Array)) {
    throw "Master item contains no bgp_peers array";
}

for (i = 0; i < root.bgp_peers.length; i++) {
    peer = root.bgp_peers[i];

    if (String(peer.site_name) === site &&
            String(peer.peer_address) === peerAddress &&
            String(peer.instance) === instance) {
        if (peer.status === null || peer.status === undefined) {
            throw "No BGP status sample";
        }
        return Number(peer.status);
    }
}

throw "Requested S2S BGP peer was not found";
```

Use `Custom on fail: Discard value`.

Recommended value map:

```text
0 = Not connected
1 = Connected
```

## Routes learned by Azure from the site

| Field | Value |
|---|---|
| Name | `{#SITE} {#INSTANCE}: Routes learned from {#BGPPEER}` |
| Type | Dependent item |
| Key | `azure.vwan.s2s.bgp.routes.learned[{#SITE},{#BGPPEER},{#INSTANCE}]` |
| Type | Numeric unsigned |
| Units | `routes` |
| History | `30d` |
| Trends | `365d` |

Preprocessing:

```javascript
var root = JSON.parse(value);
var site = "{#SITE}";
var peerAddress = "{#BGPPEER}";
var instance = "{#INSTANCE}";
var i;
var peer;

if (!root.bgp_peers || !(root.bgp_peers instanceof Array)) {
    throw "Master item contains no bgp_peers array";
}

for (i = 0; i < root.bgp_peers.length; i++) {
    peer = root.bgp_peers[i];

    if (String(peer.site_name) === site &&
            String(peer.peer_address) === peerAddress &&
            String(peer.instance) === instance) {
        if (peer.routes_learned === null ||
                peer.routes_learned === undefined) {
            throw "No routes-learned sample";
        }
        return Number(peer.routes_learned);
    }
}

throw "Requested S2S BGP peer was not found";
```

Use `Custom on fail: Discard value`.

## Routes advertised by Azure to the site

| Field | Value |
|---|---|
| Name | `{#SITE} {#INSTANCE}: Routes advertised to {#BGPPEER}` |
| Type | Dependent item |
| Key | `azure.vwan.s2s.bgp.routes.advertised[{#SITE},{#BGPPEER},{#INSTANCE}]` |
| Type | Numeric unsigned |
| Units | `routes` |
| History | `30d` |
| Trends | `365d` |

Preprocessing:

```javascript
var root = JSON.parse(value);
var site = "{#SITE}";
var peerAddress = "{#BGPPEER}";
var instance = "{#INSTANCE}";
var i;
var peer;

if (!root.bgp_peers || !(root.bgp_peers instanceof Array)) {
    throw "Master item contains no bgp_peers array";
}

for (i = 0; i < root.bgp_peers.length; i++) {
    peer = root.bgp_peers[i];

    if (String(peer.site_name) === site &&
            String(peer.peer_address) === peerAddress &&
            String(peer.instance) === instance) {
        if (peer.routes_advertised === null ||
                peer.routes_advertised === undefined) {
            throw "No routes-advertised sample";
        }
        return Number(peer.routes_advertised);
    }
}

throw "Requested S2S BGP peer was not found";
```

Use `Custom on fail: Discard value`.

## Recommended BGP tags

| Tag | Value |
|---|---|
| `component` | `bgp` |
| `site` | `{#SITE}` |
| `peer` | `{#BGPPEER}` |
| `instance` | `{#INSTANCE}` |
| `azure-bgp-ip` | `{#AZUREBGPIP}` |
| `provider` | `{#PROVIDER}` |

---

# BGP trigger prototypes

## Peer down

```text
Name: {#SITE} {#INSTANCE}: BGP peer {#BGPPEER} is down
Severity: High
Expression:
max(/Azure vWAN WEU VPN Gateway/azure.vwan.s2s.bgp.status[{#SITE},{#BGPPEER},{#INSTANCE}],15m)<1
```

## No Azure BGP telemetry

```text
Name: {#SITE} {#INSTANCE}: No Azure BGP telemetry
Severity: Warning
Expression:
nodata(/Azure vWAN WEU VPN Gateway/azure.vwan.s2s.bgp.status[{#SITE},{#BGPPEER},{#INSTANCE}],20m)=1
```

Keep these conditions separate. A reported BGP-down condition and missing telemetry require different troubleshooting.

Do not alert because a Check Point peer has zero active routes. A redundant BGP session can remain Established and healthy while another path wins route selection.

---

# BGP graph prototype

Create under the correlated S2S BGP discovery rule:

```text
Name: {#SITE} {#INSTANCE}: BGP routes
```

Add:

1. `{#SITE} {#INSTANCE}: Routes learned from {#BGPPEER}`
2. `{#SITE} {#INSTANCE}: Routes advertised to {#BGPPEER}`

Do not mix the `0/1` status item with route counts on the same graph.

---

# Expected HQ result

The master output and discovered items should show:

```text
gw-hq-dk / Instance0
  Remote public IP:    195.93.192.4
  Check Point BGP VIP: 169.254.230.4
  Check Point ASN:     65230
  Azure local BGP IP:  10.248.0.12
  Link provider:       ISP
  Link speed:          1000 Mbps
  BGP status:          1

gw-hq-dk / Instance1
  Remote public IP:    195.93.192.4
  Check Point BGP VIP: 169.254.230.4
  Check Point ASN:     65230
  Azure local BGP IP:  10.248.0.13
  Link provider:       ISP
  Link speed:          1000 Mbps
  BGP status:          1
```

Expected items per instance:

- Check Point to Azure traffic
- Azure to Check Point traffic
- Ingress packet drops
- Egress packet drops
- BGP status
- Routes learned
- Routes advertised

---

# Validation procedure

1. Test the master item before saving.
2. Confirm:
   - `tunnel_count > 0`
   - `connected_site_link_count > 0`
   - `bgp_peer_count > 0`
3. Confirm `gw-hq-dk` appears twice in `bgp_peers`.
4. Confirm:
   - Instance0 maps to `10.248.0.12`
   - Instance1 maps to `10.248.0.13`
   - Both use BGP peer `169.254.230.4`
   - Both use ASN `65230`
5. Inspect `unmatched_bgp_peers` and confirm unrelated gateway or NVA BGP series are not present in S2S discovery.
6. Execute the master item manually.
7. Wait for the two dependent discovery rules to process the value.
8. Check `Monitoring -> Latest data`.
9. Verify at least three polling cycles, approximately 15 minutes.
10. Compare `gw-hq-dk` discovered traffic with the old fixed items before disabling the old setup.

---

# Troubleshooting

| Symptom | Likely cause or check |
|---|---|
| Token request HTTP 400 | Wrong tenant ID, client ID, expired secret, or malformed secret |
| Metrics request HTTP 401 | Invalid or expired token |
| Azure request HTTP 403 | Reader role missing at the required scope |
| Script timeout | Zabbix server/proxy cannot reach Azure, DNS issue, or timeout too low |
| `connected_site_link_count = 0` | Gateway `remoteVpnSite` IDs do not match sites returned from the resource group |
| `bgp_peer_count = 0` | Site BGP peer addresses do not match Azure Monitor `BgpPeerAddress` values |
| HQ appears only once | Only one `169.254.230.4` metric series was returned for the time window |
| Wrong Azure local BGP IP | Inspect `properties.bgpSettings.bgpPeeringAddresses` on the VPN Gateway |
| Drop items have no new value | Azure did not return a current drop sample; custom failure discards it instead of recording false zero |
| LLD unsupported | Test the discovery JavaScript against the latest master value |
| Duplicate items | Old fixed HQ items remain enabled after discovery was introduced |
| BGP route values differ from Check Point | Azure and Check Point count routes received in opposite directions |
| Instance1 carries less traffic | Flow-based ECMP does not guarantee equal byte distribution |
| `.13` has zero active Check Point routes | Valid when `.12` wins route selection; do not treat as a BGP failure |

---

# Operational notes

## Direction of traffic metrics

From Azure's perspective:

- `TunnelIngressBytes`: Check Point/on-premises to Azure
- `TunnelEgressBytes`: Azure to Check Point/on-premises

## Packet drops

The raw Azure packet-drop metric may remain at a fixed non-zero value for many polling cycles. For example, `9260, 9260, 9260` creates a flat graph even when there are no new drops. Keep the raw value for diagnostics, but use a delta item for operational graphs and alerts.

### New packet-drop delta item prototypes

Create two additional dependent item prototypes under `Azure vWAN VPN tunnel discovery`. Each delta item depends on the corresponding discovered raw packet-drop item prototype.

#### New ingress packet drops

| Field | Value |
|---|---|
| Name | `{#CONNECTION} {#INSTANCE}: New ingress packet drops` |
| Type | Dependent item |
| Key | `azure.vwan.vpn.ingress.drops.delta[{#CONNECTION},{#INSTANCE},{#REMOTEIP}]` |
| Master item | `{#CONNECTION} {#INSTANCE}: Ingress packet drops` |
| Type of information | Numeric float |
| Units | `packets` |
| History | `30d` |
| Trends | `365d` |

Preprocessing steps, in this exact order:

1. `Simple change`
2. `JavaScript`

```javascript
var v = Number(value);

if (isNaN(v)) {
    throw "Invalid packet-drop delta";
}

if (v < 0) {
    return 0;
}

return v;
```

The second step suppresses a false negative spike if Azure resets the raw counter.

#### New egress packet drops

| Field | Value |
|---|---|
| Name | `{#CONNECTION} {#INSTANCE}: New egress packet drops` |
| Type | Dependent item |
| Key | `azure.vwan.vpn.egress.drops.delta[{#CONNECTION},{#INSTANCE},{#REMOTEIP}]` |
| Master item | `{#CONNECTION} {#INSTANCE}: Egress packet drops` |
| Type of information | Numeric float |
| Units | `packets` |
| History | `30d` |
| Trends | `365d` |

Use the same preprocessing sequence:

1. `Simple change`
2. `JavaScript` with the reset-handling code above.

Example:

```text
Raw Azure counter:  9260  9260  9278  9278
Stored delta:          0     18     0
```

The first sample establishes the baseline and therefore has no delta.

### Packet-drop graph prototype

Do not graph the raw counters for health visualization. Create or update the graph prototype to use:

- `{#CONNECTION} {#INSTANCE}: New ingress packet drops`
- `{#CONNECTION} {#INSTANCE}: New egress packet drops`

Suggested graph name:

```text
{#CONNECTION} {#INSTANCE}: New VPN packet drops
```

A healthy tunnel should normally stay at zero and show spikes only when the raw Azure counter increases. Keep the raw packet-drop items in Latest data for diagnostics.

### Packet-drop triggers

Triggers should use the delta items, not the raw counters. A raw value such as `9260` can remain unchanged indefinitely and must not generate a permanent alarm.

Start with Warning severity and a baseline threshold such as more than `100` newly dropped packets in 15 minutes:

```text
sum(/Azure vWAN WEU VPN Gateway/azure.vwan.vpn.ingress.drops.delta[{#CONNECTION},{#INSTANCE},{#REMOTEIP}],15m)>100
```

```text
sum(/Azure vWAN WEU VPN Gateway/azure.vwan.vpn.egress.drops.delta[{#CONNECTION},{#INSTANCE},{#REMOTEIP}],15m)>100
```

Tune this threshold after one or two weeks of baseline data. If isolated rekey or convergence events produce no meaningful user impact, increase the threshold rather than alerting on every single packet.

## BGP route direction

- `BgpRoutesLearned`: routes Azure receives from the remote site
- `BgpRoutesAdvertised`: routes Azure advertises to the remote site
- Check Point `show bgp peers` `Routes`: routes Check Point receives from Azure

Therefore, the Azure and Check Point route counts are not expected to be equal.

## ClusterXL behavior

Azure monitors `169.254.230.4`, the BGP cluster VIP. A failover from member `.5` to member `.6` should not create a new Azure BGP monitoring identity. Member role and ClusterXL state should be monitored separately on Check Point.

## Discovery stability

The master uses a six-hour lookback to retain quiet tunnel and BGP series. Both LLD rules use `Discard unchanged with heartbeat: 1h`, while dependent metric items continue to update every five minutes.

## API call pattern

Each five-minute master poll performs:

1. One Entra token request
2. One VPN Gateway configuration request
3. One or more paged VPN-site list requests
4. One tunnel metrics request
5. One BGP-status metrics request
6. One BGP-route metrics request

This is substantially more efficient than separate requests per site, tunnel, and item.

---

# Migration and cleanup

After at least three successful discovery cycles:

1. Compare old fixed `gw-hq-dk` items with newly discovered items.
2. Confirm traffic direction and scale.
3. Confirm both HQ BGP instances report status `1`.
4. Disable the old fixed master and dependent items.
5. Keep the old items disabled for one day as rollback protection.
6. Delete the old fixed items after validation.
7. Remove obsolete macros only if no remaining item uses them:
   - `{$AZURE.VPN.CONNECTION}`
   - `{$AZURE.VPN.REMOTE.IP}`
8. Keep the four general Azure authentication/resource macros.

---

# Security recommendations

- Store the application secret only as a Zabbix Secret text macro.
- Scope the service principal to Reader at the narrowest practical Azure scope that still covers the gateway, VPN sites, and metrics.
- Rotate the client secret according to the organization's credential policy.
- Restrict access to the Zabbix host/template and master item because test output can contain internal IP addresses, site names, ASNs, and link details.
- Do not log or expose the OAuth access token.
- Consider replacing client secrets with certificate-based authentication in a future revision if supported by the chosen Zabbix execution design.

---

# References

- Microsoft Learn: VPN Sites, List by Resource Group  
  https://learn.microsoft.com/en-us/rest/api/virtualwan/vpn-sites/list-by-resource-group
- Microsoft Learn: Supported metrics for `Microsoft.Network/vpnGateways`  
  https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-network-vpngateways-metrics
- Microsoft Learn: Azure Monitor Metrics REST API  
  https://learn.microsoft.com/en-us/rest/api/monitor/metrics/list
- Zabbix 7.4: Low-level discovery  
  https://www.zabbix.com/documentation/7.4/en/manual/discovery/low_level_discovery
- Zabbix 7.4: Item prototypes  
  https://www.zabbix.com/documentation/7.4/en/manual/discovery/low_level_discovery/item_prototypes
- Check Point: Azure Virtual WAN interoperability, `sk176249`  
  https://support.checkpoint.com/results/sk/sk176249

---

# Why Zabbix tunnel bandwidth can differ from the Azure Portal gateway graph

A difference between the Zabbix per-tunnel graph and the Azure Portal **S2S Gateway Average Bandwidth** graph is expected. The two views answer different questions.

## What Zabbix stores

The tunnel discovery uses the Azure Monitor metrics:

```text
TunnelIngressBytes
TunnelEgressBytes
```

for every `ConnectionName + RemoteIP + Instance` five-minute bucket. Zabbix converts each directional five-minute byte total into an average bit rate:

```text
bps = bytes * 8 / 300
```

This gives separate values for:

```text
gw-hq-dk Instance0: Check Point -> Azure
gw-hq-dk Instance0: Azure -> Check Point
gw-hq-dk Instance1: Check Point -> Azure
gw-hq-dk Instance1: Azure -> Check Point
```

The calculation was verified against Azure's native `TunnelAverageBandwidth` metric. The values match extremely closely, proving that the Zabbix conversion and per-instance graph are correct.

## Verified example at 2026-10-30 06:41 UTC

### Instance1

Azure Monitor byte metrics returned approximately:

```text
TunnelIngressBytes -> 1.142091 Mbps
TunnelEgressBytes  -> 401.023594 Mbps
```

Combined throughput for the tunnel instance:

```text
1.142091 + 401.023594 = 402.165685 Mbps
```

Azure native `TunnelAverageBandwidth` returned:

```text
402.165680 Mbps
```

The values are effectively identical apart from rounding.

### Instance0

Azure Monitor byte metrics returned approximately:

```text
TunnelIngressBytes -> 0.562524 Mbps
TunnelEgressBytes  -> 141.722047 Mbps
```

Combined throughput:

```text
0.562524 + 141.722047 = 142.284571 Mbps
```

Azure native `TunnelAverageBandwidth` returned:

```text
142.284568 Mbps
```

Again, the values are effectively identical.

## What the Azure Portal gateway graph displays

For the same timestamp, Azure's gateway-level `AverageBandwidth` metric returned:

```text
272.225292 Mbps
```

This corresponds to the average of the two VPN Gateway instance tunnel bandwidth values:

```text
(402.165680 + 142.284568) / 2
= 272.225124 Mbps
```

The very small difference from `272.225292 Mbps` is rounding/metric precision.

Therefore, for this scale-unit-2 VPN Gateway:

```text
Azure Gateway AverageBandwidth
approximately equals
(Instance0 TunnelAverageBandwidth + Instance1 TunnelAverageBandwidth) / 2
```

It is **not** the maximum busy-instance throughput and it is **not** the sum of both instances.

## Operational interpretation

During the verified example:

```text
Instance0 total throughput: approximately 142 Mbps
Instance1 total throughput: approximately 402 Mbps
Actual combined traffic:    approximately 544 Mbps
Azure gateway average:      approximately 272 Mbps
```

Therefore, seeing approximately `35 MB/s` / `272-280 Mbps` in the Azure Portal while Zabbix shows an individual tunnel instance near `400 Mbps` does not indicate an error.

Zabbix intentionally keeps the more granular per-instance, per-direction view. This is more useful for operational troubleshooting because it shows:

- which VPN Gateway instance is carrying the traffic;
- traffic direction independently;
- flow-distribution imbalance between Instance0 and Instance1;
- whether one tunnel is idle while the other tunnel carries most traffic;
- the actual throughput handled by a particular tunnel instance rather than only a gateway average.

No configuration change is required. The existing Zabbix traffic calculation should remain in place.

## Comparison summary

| Measurement | Scope | Direction | Example at 06:41 UTC |
|---|---|---|---:|
| Zabbix Instance1 egress | `gw-hq-dk` / Instance1 | Azure -> Check Point | ~401.02 Mbps |
| Zabbix Instance1 ingress | `gw-hq-dk` / Instance1 | Check Point -> Azure | ~1.14 Mbps |
| Native TunnelAverageBandwidth Instance1 | `gw-hq-dk` / Instance1 | Combined | ~402.17 Mbps |
| Native TunnelAverageBandwidth Instance0 | `gw-hq-dk` / Instance0 | Combined | ~142.28 Mbps |
| Azure Portal Gateway AverageBandwidth | Whole VPN Gateway | Average across gateway instances | ~272.23 Mbps |

When validating future discrepancies, compare the Zabbix sum of ingress plus egress for a specific tunnel instance with Azure `TunnelAverageBandwidth` for that same `ConnectionName`, `RemoteIP`, `Instance`, five-minute timestamp, and aggregation interval. Compare the Azure Portal `AverageBandwidth` only against the gateway-level average, not against one individual tunnel instance.
