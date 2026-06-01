What I Now Understand — Confirm Before I Begin

Let me restate everything precisely:

KEY VAULT (Tunnel Sub) — 3 Objects Confirmed:

├── KEYS section:        EMPTY (nothing here)
│
├── SECRETS section:
│     └── Company CA Trust Chain
│           ├── Root CA cert
│           └── Intermediate CA cert
│           (PEM bundle your internal PKI issued)
│
└── CERTIFICATES section:
      └── Pulsar Certificates
            └── This is likely the wildcard TLS cert
                *.yourdomain.com  OR  pulsar.yourdomain.com
                Signed by your Company CA


My working assumption of the full trust model:

Company Internal CA (root of trust)
        │
        ├── Signs ──► Pulsar TLS Cert (stored in KV Certificates)
        │              Used by: Pulsar Proxy/Broker to present identity
        │
        └── Signs ──► App Client Cert (if mTLS)
                       Used by: App Pod to present identity to Pulsar

Company CA Trust Chain (stored in KV Secrets)
        │
        ├── Mounted into ──► Company AKS App Pods
        │                     So app TRUSTS Pulsar cert
        │
        └── Delivered to ──► Stream Native AKS
                              So Pulsar TRUSTS app client cert


My 5 Remaining Assumptions (tell me if wrong):



|# |Assumption                                                                                                                       |Confidence               |
|--|---------------------------------------------------------------------------------------------------------------------------------|-------------------------|
|Q1|Company AKS uses Azure Private DNS Resolver OR CoreDNS custom forwarder to resolve Pulsar DNS through Tunnel Sub Private DNS Zone|High                     |
|Q2|Private Endpoint in Tunnel Sub points at Stream Native PLS in front of their Internal LB                                         |Confirmed by you         |
|Q3|CA Trust Chain delivered via CSI Secret Store Driver or cert-manager into pods                                                   |Unknown — I’ll cover both|
|Q4|Auth = mTLS + possibly JWT token — Pulsar client cert OR token stored in KV                                                      |Will cover both          |
|Q5|Company AKS uses Azure CNI — pods get real VNet IPs                                                                              |Assuming prod default    |

FULL PRODUCTION WALKTHROUGH

ARCHITECTURE — COMPLETE PICTURE

╔══════════════════════════════════════════════════════════════════════════════════╗
║                         COMPANY APP SUBSCRIPTION                                ║
║                                                                                  ║
║  ┌─────────────────────────────────────────────────────────────┐                ║
║  │                    Company Hub VNet                          │                ║
║  │   Address Space: 10.0.0.0/8 (example)                       │                ║
║  │                                                              │                ║
║  │  ┌──────────────────────────────────────────────────────┐   │                ║
║  │  │              AKS Node Subnet                          │   │                ║
║  │  │              e.g. 10.10.1.0/24                        │   │                ║
║  │  │                                                        │   │                ║
║  │  │  ┌─────────────────┐    ┌─────────────────┐          │   │                ║
║  │  │  │  App Pod        │    │  App Pod        │          │   │                ║
║  │  │  │  (Order Svc)    │    │  (Payment Svc)  │          │   │                ║
║  │  │  │                 │    │                 │          │   │                ║
║  │  │  │  Mounts from KV:│    │  Mounts from KV:│          │   │                ║
║  │  │  │  - CA chain     │    │  - CA chain     │          │   │                ║
║  │  │  │  - Client cert  │    │  - Client cert  │          │   │                ║
║  │  │  │  or JWT token   │    │  or JWT token   │          │   │                ║
║  │  │  └────────┬────────┘    └────────┬────────┘          │   │                ║
║  │  │           │                      │                    │   │                ║
║  │  │           └──────────┬───────────┘                    │   │                ║
║  │  │                      │ DNS: pulsar.yourdomain.com      │   │                ║
║  │  │                      │ Resolves to: 10.20.2.10         │   │                ║
║  │  │                      │ (Private Endpoint IP)           │   │                ║
║  │  └──────────────────────┼────────────────────────────────┘   │                ║
║  │                         │                                      │                ║
║  │  NSG on AKS Subnet:     │                                      │                ║
║  │  OUTBOUND 6651 ALLOW    │                                      │                ║
║  │  OUTBOUND 443  ALLOW    │                                      │                ║
║  └─────────────────────────┼────────────────────────────────────┘                ║
║                            │                                                      ║
║                    Hub VNet Peering                                               ║
║                    (Allow forwarded traffic)                                      ║
║                    (Allow gateway transit)                                        ║
╚════════════════════════════╪═════════════════════════════════════════════════════╝
                             │
                    VNet Peering Link
                    Bidirectional
                    10.0.0.0/8 ◄──► 10.20.0.0/16
                             │
╔════════════════════════════╪═════════════════════════════════════════════════════╗
║                         TUNNEL SUBSCRIPTION                                      ║
║                                                                                  ║
║  ┌─────────────────────────────────────────────────────────────┐                ║
║  │                    Tunnel VNet                               │                ║
║  │   Address Space: 10.20.0.0/16 (example)                     │                ║
║  │                                                              │                ║
║  │  ┌──────────────────────────────────────────────────────┐   │                ║
║  │  │         Private Endpoint Subnet                       │   │                ║
║  │  │         e.g. 10.20.2.0/24                             │   │                ║
║  │  │         (Private Endpoint Policies DISABLED)          │   │                ║
║  │  │                                                        │   │                ║
║  │  │  ┌─────────────────────────────────────────────┐     │   │                ║
║  │  │  │         Private Endpoint                     │     │   │                ║
║  │  │  │         Name: pe-pulsar-streamnative          │     │   │                ║
║  │  │  │         Private IP: 10.20.2.10               │     │   │                ║
║  │  │  │         Status: Approved                     │     │   │                ║
║  │  │  │         Target: Stream Native PLS            │     │   │                ║
║  │  │  │                                              │     │   │                ║
║  │  │  │  ┌─────────────────────────────────┐        │     │   │                ║
║  │  │  │  │  NIC (Network Interface)        │        │     │   │                ║
║  │  │  │  │  IP: 10.20.2.10                 │        │     │   │                ║
║  │  │  │  │  Subnet: PE Subnet              │        │     │   │                ║
║  │  │  │  └─────────────────────────────────┘        │     │   │                ║
║  │  │  └─────────────────────────────────────────────┘     │   │                ║
║  │  └──────────────────────────────────────────────────────┘   │                ║
║  │                                                              │                ║
║  │  NSG on PE Subnet:                                           │                ║
║  │  INBOUND  from Company AKS subnet — 6651 ALLOW               │                ║
║  │  INBOUND  from Company AKS subnet — 443  ALLOW               │                ║
║  │  OUTBOUND to Stream Native VNet  — 6651 ALLOW                │                ║
║  │                                                              │                ║
║  │  ┌──────────────────────────────────────────────────────┐   │                ║
║  │  │         Private DNS Zone                              │   │                ║
║  │  │         *.yourdomain.com   OR                         │   │                ║
║  │  │         pulsar.yourdomain.com                         │   │                ║
║  │  │                                                        │   │                ║
║  │  │  A Record:                                             │   │                ║
║  │  │  pulsar.yourdomain.com → 10.20.2.10                   │   │                ║
║  │  │                                                        │   │                ║
║  │  │  VNet Links:                                           │   │                ║
║  │  │  ├── Link to Tunnel VNet      ✓                        │   │                ║
║  │  │  └── Link to Company AKS VNet ✓ ← CRITICAL            │   │                ║
║  │  └──────────────────────────────────────────────────────┘   │                ║
║  │                                                              │                ║
║  │  ┌──────────────────────────────────────────────────────┐   │                ║
║  │  │         Key Vault                                     │   │                ║
║  │  │         Name: kv-tunnel-prod                          │   │                ║
║  │  │                                                        │   │                ║
║  │  │  KEYS:         (empty)                                 │   │                ║
║  │  │  SECRETS:      company-ca-trust-chain                  │   │                ║
║  │  │  CERTIFICATES: pulsar-tls-cert                         │   │                ║
║  │  └──────────────────────────────────────────────────────┘   │                ║
║  └─────────────────────────────────────────────────────────────┘                ║
║                                                                                  ║
╚══════════════════════════════════════════════════════════════════════════════════╝
                             │
                    Azure Private Link
                    (Microsoft backbone)
                    No public internet
                    No VNet peering needed
                    to Stream Native Sub
                             │
╔════════════════════════════╪═════════════════════════════════════════════════════╗
║                    STREAM NATIVE SUBSCRIPTION                                    ║
║                                                                                  ║
║  ┌─────────────────────────────────────────────────────────────┐                ║
║  │                 Stream Native VNet                           │                ║
║  │                                                              │                ║
║  │  ┌──────────────────────────────────────────────────────┐   │                ║
║  │  │         Private Link Service (PLS)                    │   │                ║
║  │  │         Sits in front of Internal LB                  │   │                ║
║  │  │         Accepts connections from:                     │   │                ║
║  │  │         → Your Private Endpoint                       │   │                ║
║  │  │         NAT IP range: e.g. 172.16.0.0/24             │   │                ║
║  │  └──────────────────────┬───────────────────────────────┘   │                ║
║  │                         │                                    │                ║
║  │  ┌──────────────────────▼───────────────────────────────┐   │                ║
║  │  │         Internal Load Balancer (ILB)                  │   │                ║
║  │  │         Frontend IP: PLS NAT range                    │   │                ║
║  │  │         Backend Pool: Pulsar Proxy Pods               │   │                ║
║  │  │         Port: 6651 (TLS) or 6650 (plain)             │   │                ║
║  │  └──────────────────────┬───────────────────────────────┘   │                ║
║  │                         │                                    │                ║
║  │  ┌──────────────────────▼───────────────────────────────┐   │                ║
║  │  │         Pulsar Proxy (inside AKS)                     │   │                ║
║  │  │         Kubernetes Service Type: LoadBalancer (ILB)   │   │                ║
║  │  │         Presents: Pulsar TLS Cert from KV             │   │                ║
║  │  │         Validates: Company CA chain                   │   │                ║
║  │  └──────────────────────┬───────────────────────────────┘   │                ║
║  │                         │                                    │                ║
║  │  ┌──────────────────────▼───────────────────────────────┐   │                ║
║  │  │         Pulsar Broker Pods                            │   │                ║
║  │  │         ├── broker-0                                  │   │                ║
║  │  │         ├── broker-1                                  │   │                ║
║  │  │         └── broker-2                                  │   │                ║
║  │  └──────────────────────┬───────────────────────────────┘   │                ║
║  │                         │                                    │                ║
║  │  ┌──────────────────────▼───────────────────────────────┐   │                ║
║  │  │         BookKeeper (Storage Layer)                    │   │                ║
║  │  │         ├── bookie-0                                  │   │                ║
║  │  │         ├── bookie-1                                  │   │                ║
║  │  │         └── bookie-2                                  │   │                ║
║  │  └──────────────────────────────────────────────────────┘   │                ║
║  └─────────────────────────────────────────────────────────────┘                ║
╚══════════════════════════════════════════════════════════════════════════════════╝


PORTAL WALKTHROUGH — STEP BY STEP

STEP 1 — Start in Company App Subscription → AKS Cluster

Azure Portal
→ Subscriptions → [Company App Sub]
→ Kubernetes Services → [your-company-aks]
→ Settings → Properties


Check:



|Field             |What to look for                  |Why it matters                                       |
|------------------|----------------------------------|-----------------------------------------------------|
|Kubernetes version|1.27+ recommended                 |Older versions have CNI bugs                         |
|Network plugin    |azure (Azure CNI)                 |Pods get real VNet IPs — NSG rules apply at pod level|
|Network policy    |azure or calico                   |Controls pod-to-pod traffic                          |
|Private cluster   |Enabled                           |API server not exposed to internet                   |
|Outbound type     |loadBalancer or userDefinedRouting|UDR means custom routes needed                       |

→ Settings → Networking


Check:



|Field         |Expected value                                     |
|--------------|---------------------------------------------------|
|VNet          |company-hub-vnet                                   |
|Node subnet   |aks-node-subnet (10.10.1.0/24)                     |
|Service CIDR  |10.100.0.0/16 (must not overlap anything)          |
|DNS service IP|10.100.0.10                                        |
|Pod CIDR      |(Azure CNI — no pod CIDR, pods use node subnet IPs)|

kubectl equivalent — run from your test pod:

# Get AKS cluster networking info
kubectl get nodes -o wide
# Shows NODE IPs — these should be in your AKS node subnet

kubectl describe node <node-name> | grep -i cidr
# Shows pod CIDR per node

kubectl get svc -n kube-system
# Shows coredns, kube-dns services


STEP 2 — Check CoreDNS Config in Company AKS

This is the most critical step for Pulsar connectivity. If DNS is wrong nothing works.

Azure Portal → AKS → [your-company-aks]
→ Run command  (or use kubectl)


kubectl command:

kubectl get configmap coredns-custom -n kube-system -o yaml


What you MUST see — a custom forwarder for your Pulsar domain:

apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  yourdomain.server: |
    yourdomain.com:53 {
        forward . 168.63.129.16
    }


This tells CoreDNS: “For anything ending in yourdomain.com, forward the DNS query to Azure’s internal DNS resolver 168.63.129.16 — which will then look it up in the Private DNS Zone in Tunnel Sub.”

If this configmap is MISSING or WRONG → app pod cannot resolve pulsar.yourdomain.com → connection refused → Pulsar producer/consumer fails immediately.

Verify DNS resolution FROM a test pod:

# Exec into your test pod or any app pod
kubectl exec -it <test-pod> -n <namespace> -- /bin/sh

# Run nslookup
nslookup pulsar.yourdomain.com

# Expected output:
# Server:    10.100.0.10       ← CoreDNS service IP
# Address:   10.100.0.10:53
#
# Name:      pulsar.yourdomain.com
# Address:   10.20.2.10        ← Private Endpoint IP in Tunnel Sub

# If you get:
# ** server can't find pulsar.yourdomain.com: NXDOMAIN
# → DNS is broken. Check CoreDNS configmap and Private DNS Zone VNet link.


# Also run dig for more detail
dig pulsar.yourdomain.com

# Check what DNS server answered
dig pulsar.yourdomain.com +short
# Should return: 10.20.2.10


STEP 3 — Check NSG on Company AKS Node Subnet

Azure Portal
→ Subscriptions → [Company App Sub]
→ Virtual Networks → company-hub-vnet
→ Subnets → aks-node-subnet
→ Network Security Group → [aks-nsg]
→ Outbound security rules


Must have these outbound rules:



|Priority|Name             |Port|Protocol|Destination                    |Action|
|--------|-----------------|----|--------|-------------------------------|------|
|100     |Allow-Pulsar-TLS |6651|TCP     |10.20.2.0/24 (Tunnel PE subnet)|Allow |
|110     |Allow-HTTPS      |443 |TCP     |10.20.2.0/24                   |Allow |
|200     |Allow-KV-HTTPS   |443 |TCP     |Key Vault private IP           |Allow |
|65000   |AllowVnetOutbound|Any |Any     |VirtualNetwork                 |Allow |
|65500   |DenyAllOutbound  |Any |Any     |Any                            |Deny  |

kubectl equivalent:

# Check if outbound traffic is blocked from a pod
kubectl exec -it <test-pod> -n <namespace> -- /bin/sh

# Test TCP connectivity to Private Endpoint IP on Pulsar port
nc -zv 10.20.2.10 6651
# Expected: Connection succeeded
# If: Connection timed out → NSG blocking OR route missing

# Test HTTPS to Key Vault
nc -zv <keyvault-private-ip> 443


STEP 4 — Check VNet Peering (Company Sub → Tunnel Sub)

Azure Portal
→ Subscriptions → [Company App Sub]
→ Virtual Networks → company-hub-vnet
→ Peerings


What you must see:



|Field                  |Expected                              |
|-----------------------|--------------------------------------|
|Peering name           |peer-company-to-tunnel                |
|Peering status         |Connected ← MUST be this              |
|Allow forwarded traffic|Enabled                               |
|Allow gateway transit  |Enabled or Disabled (depends on setup)|
|Use remote gateways    |Disabled (usually)                    |
|Remote VNet            |tunnel-vnet (Tunnel Sub)              |

Now check the OTHER SIDE — go to Tunnel Sub:

Azure Portal
→ Subscriptions → [Tunnel Sub]
→ Virtual Networks → tunnel-vnet
→ Peerings




|Field                  |Expected                                  |
|-----------------------|------------------------------------------|
|Peering name           |peer-tunnel-to-company                    |
|Peering status         |Connected ← BOTH sides must show Connected|
|Allow forwarded traffic|Enabled                                   |

Common break: One side shows Disconnected or Initiated — this means peering is broken and traffic cannot flow from Company AKS to Tunnel Sub PE.

STEP 5 — Move to Tunnel Subscription → Private DNS Zone

Azure Portal
→ Subscriptions → [Tunnel Sub]
→ Private DNS Zones → [yourdomain.com or *.yourdomain.com]


Check A Records:

→ Overview → Record sets




|Name  |Type|TTL|IP Address|
|------|----|---|----------|
|pulsar|A   |300|10.20.2.10|
|@     |A   |300|10.20.2.10|

The IP must match the Private Endpoint NIC IP exactly.

Check VNet Links:

→ Settings → Virtual network links




|Link name           |VNet            |Registration|Status  |
|--------------------|----------------|------------|--------|
|link-to-tunnel-vnet |tunnel-vnet     |Disabled    |Linked ✓|
|link-to-company-vnet|company-hub-vnet|Disabled    |Linked ✓|

If link-to-company-vnet is missing: Company AKS pods cannot resolve the DNS name. This is the #1 production issue we see with Private DNS Zones.

STEP 6 — Check Private Endpoint in Tunnel Sub

Azure Portal
→ Subscriptions → [Tunnel Sub]
→ Private Endpoints → [pe-pulsar-streamnative]


Overview tab — check:



|Field              |Expected value                     |
|-------------------|-----------------------------------|
|Connection state   |Approved ← CRITICAL                |
|Provisioning state |Succeeded                          |
|Resource           |Stream Native PLS resource ID      |
|Target sub-resource|(whatever SN named their PLS group)|

If Connection state = Pending: Stream Native hasn’t approved the connection. Go to their side.

If Connection state = Rejected: You need to raise a new Private Endpoint request.

→ Settings → DNS Configuration




|Field           |Expected                                              |
|----------------|------------------------------------------------------|
|Private DNS Zone|[yourdomain.com](https://yourdomain.com)              |
|FQDN            |[pulsar.yourdomain.com](https://pulsar.yourdomain.com)|
|IP address      |10.20.2.10                                            |

→ Settings → Network Interface
→ Click on the NIC name


NIC details — check:



|Field             |Expected                |
|------------------|------------------------|
|Private IP address|10.20.2.10              |
|Subnet            |pe-subnet (10.20.2.0/24)|
|Virtual network   |tunnel-vnet             |
|Attached to       |pe-pulsar-streamnative  |

STEP 7 — Check NSG on Tunnel PE Subnet

Azure Portal
→ Subscriptions → [Tunnel Sub]
→ Virtual Networks → tunnel-vnet
→ Subnets → pe-subnet
→ Network Security Group → [tunnel-pe-nsg]


Critical note: Azure does NOT enforce NSGs on Private Endpoint NICs by default. The subnet must have PrivateEndpointNetworkPolicies set to Enabled in Terraform/Portal if you want NSG rules to apply to PE traffic.

Check subnet settings:

→ Virtual Networks → tunnel-vnet → Subnets → pe-subnet
→ Private endpoint network policies




|Setting           |Your situation                            |
|------------------|------------------------------------------|
|Disabled (default)|NSG rules do NOT apply to PE NIC          |
|Enabled           |NSG rules DO apply — check rules carefully|

az CLI to check:

az network vnet subnet show \
  --resource-group <tunnel-rg> \
  --vnet-name tunnel-vnet \
  --name pe-subnet \
  --query "privateEndpointNetworkPolicies"
# Returns: "Disabled" or "Enabled"


STEP 8 — Check Key Vault in Tunnel Sub

Azure Portal
→ Subscriptions → [Tunnel Sub]
→ Key Vaults → [kv-tunnel-prod]
→ Objects → Secrets


Secrets — confirm:



|Secret name           |What it contains                     |Who reads it                                                  |
|----------------------|-------------------------------------|--------------------------------------------------------------|
|company-ca-trust-chain|PEM bundle: Root CA + Intermediate CA|Company AKS pods (via CSI or cert-manager) + Stream Native AKS|

→ Objects → Certificates




|Certificate name|What it contains                                   |Who reads it    |
|----------------|---------------------------------------------------|----------------|
|pulsar-tls-cert |TLS cert for Pulsar endpoint (signed by Company CA)|Pulsar Proxy pod|

Check access policies or RBAC:

→ Access control (IAM)  ← if using Azure RBAC
OR
→ Access policies        ← if using legacy vault access policy


Must see:



|Identity                                  |Permission                 |Purpose                               |
|------------------------------------------|---------------------------|--------------------------------------|
|Company AKS Managed Identity              |Secret Get, Certificate Get|App pods reading CA chain + creds     |
|Stream Native AKS Managed Identity (or SP)|Secret Get, Certificate Get|Pulsar reading its TLS cert + CA chain|

Check KV Private Endpoint:

→ Settings → Networking → Private endpoint connections


Key Vault itself should also have a Private Endpoint so pods access it privately — not over public internet.

kubectl — verify CSI secret mount in app pod:

kubectl exec -it <app-pod> -n <namespace> -- /bin/sh

# Check if CA trust chain is mounted
ls /mnt/secrets/  
# or wherever your CSI mount path is

cat /mnt/secrets/company-ca-trust-chain
# Should print PEM certificate chain
# -----BEGIN CERTIFICATE-----
# ...

# Verify the cert chain is valid
openssl verify -CAfile /mnt/secrets/company-ca-trust-chain \
  /mnt/secrets/pulsar-client-cert.pem
# Expected: /mnt/secrets/pulsar-client-cert.pem: OK


STEP 9 — Verify Private Link Service on Stream Native Side

This is inside Stream Native Subscription which you own.

Azure Portal
→ Subscriptions → [Stream Native Sub]
→ Private Link Services → [pls-pulsar-streamnative]


Overview — check:



|Field             |Expected                                             |
|------------------|-----------------------------------------------------|
|Provisioning state|Succeeded                                            |
|Load balancer     |[stream-native-ilb]                                  |
|Alias             |pls-xxxxx.region.azure.privatelinkservice ← copy this|
|Visibility        |Your subscription only (or specific subs)            |

Private endpoint connections tab:



|Connection            |State   |Requester |
|----------------------|--------|----------|
|pe-pulsar-streamnative|Approved|Tunnel Sub|

If your PE shows Pending here → go to this tab → select connection → Approve.

→ Settings → NAT IP configuration




|Field     |Value                            |
|----------|---------------------------------|
|NAT subnet|(dedicated NAT subnet in SN VNet)|
|NAT IPs   |e.g. 172.16.0.4, 172.16.0.5      |

The PLS uses these NAT IPs to represent your connection inside the Stream Native VNet. Pulsar sees traffic sourced from these NAT IPs — NOT your original pod IP.

STEP 10 — Check Stream Native AKS Cluster

Azure Portal
→ Subscriptions → [Stream Native Sub]
→ Kubernetes Services → [streamnative-aks]
→ Settings → Networking


Check:



|Field            |Expected         |
|-----------------|-----------------|
|Network plugin   |azure (Azure CNI)|
|Private cluster  |Enabled          |
|Load balancer SKU|Standard         |

Now go to Kubernetes workloads:

→ Workloads → Deployments or StatefulSets


Look for:

	•	pulsar-proxy deployment
	•	pulsar-broker statefulset
	•	bookkeeper statefulset
	•	zookeeper statefulset

kubectl — run against Stream Native AKS (if you have access):

# Set context to Stream Native AKS
az aks get-credentials \
  --resource-group <sn-rg> \
  --name streamnative-aks \
  --subscription <sn-sub-id>

kubectl config use-context streamnative-aks

# Check Pulsar proxy service — this is what ILB exposes
kubectl get svc -n pulsar | grep proxy
# Expected:
# pulsar-proxy   LoadBalancer   10.x.x.x   <internal-ip>   6651:xxxxx/TCP

# The LoadBalancer IP should be the ILB frontend IP
# PLS sits in front of this ILB

# Check Pulsar broker pods
kubectl get pods -n pulsar | grep broker
# Expected: all Running

# Check Pulsar proxy pods
kubectl get pods -n pulsar | grep proxy
# Expected: all Running

# Check broker logs for TLS errors
kubectl logs -n pulsar pulsar-broker-0 | grep -i "tls\|ssl\|cert\|auth" | tail -50

# Check proxy logs
kubectl logs -n pulsar pulsar-proxy-0 | grep -i "tls\|ssl\|cert\|error" | tail -50


STEP 11 — Full End-to-End Connectivity Test From App Pod

# Go back to Company AKS context
az aks get-credentials \
  --resource-group <company-rg> \
  --name company-aks \
  --subscription <company-sub-id>

kubectl config use-context company-aks

# Exec into test pod
kubectl exec -it <test-pod> -n <namespace> -- /bin/sh

# STEP 1: DNS resolution
nslookup pulsar.yourdomain.com
# Must return: 10.20.2.10

# STEP 2: TCP port test to Private Endpoint
nc -zv 10.20.2.10 6651
# Must return: succeeded

# STEP 3: TLS handshake test
openssl s_client \
  -connect pulsar.yourdomain.com:6651 \
  -CAfile /mnt/secrets/company-ca-trust-chain \
  -cert /mnt/secrets/client-cert.pem \
  -key /mnt/secrets/client-key.pem

# What to look for in output:
# SSL handshake has read XXXX bytes
# Verify return code: 0 (ok)   ← MUST see this
#
# If you see:
# Verify return code: 21 (unable to verify the first certificate)
# → CA chain is wrong or missing

# STEP 4: Pulsar client test (if pulsar-client installed in test pod)
pulsar-client \
  --url pulsar+ssl://pulsar.yourdomain.com:6651 \
  --tlsTrustCertsFilePath /mnt/secrets/company-ca-trust-chain \
  --tlsAllowInsecureConnection false \
  produce persistent://public/default/test-topic \
  -m "hello from company aks"


COMPLETE TROUBLESHOOTING CHECKLIST

DNS LAYER
□ CoreDNS configmap coredns-custom has forwarder for yourdomain.com
□ Private DNS Zone has A record pointing to PE IP
□ Private DNS Zone has VNet link to Company AKS VNet
□ Private DNS Zone has VNet link to Tunnel VNet
□ nslookup pulsar.yourdomain.com from test pod returns correct IP
□ No split-brain DNS (same name resolving differently from different pods)

NETWORK LAYER
□ VNet Peering Company→Tunnel status = Connected
□ VNet Peering Tunnel→Company status = Connected (both sides)
□ Allow forwarded traffic = Enabled on both peerings
□ NSG on AKS node subnet allows outbound 6651 to Tunnel PE subnet
□ NSG on Tunnel PE subnet (if policies enabled) allows inbound from Company
□ No UDR routing Pulsar traffic to a firewall that blocks it
□ Private Endpoint connection state = Approved
□ Private Endpoint NIC IP matches DNS A record exactly

TLS / CERTIFICATE LAYER
□ Company CA trust chain is present in Key Vault Secrets
□ Pulsar TLS cert is present in Key Vault Certificates
□ Pulsar TLS cert is signed by Company CA (verify with openssl)
□ CA chain is mounted into app pods (check /mnt/secrets or equiv)
□ CA chain is loaded into Pulsar Proxy trust store
□ TLS handshake completes: openssl s_client verify return code = 0
□ Cert expiry: openssl x509 -in cert.pem -noout -dates

STREAM NATIVE PLS LAYER
□ PLS provisioning state = Succeeded
□ PLS has Private Endpoint connection from Tunnel Sub = Approved
□ PLS frontend is attached to Stream Native Internal LB
□ Internal LB backend pool contains Pulsar Proxy pods
□ Pulsar Proxy pods are Running and Ready
□ Pulsar Broker pods are Running and Ready

KEY VAULT ACCESS LAYER
□ Company AKS Managed Identity has Secret Get on KV
□ KV has private endpoint (not accessed over public internet)
□ CSI SecretStore or cert-manager is running in Company AKS
□ SecretProviderClass resource exists and references correct KV secrets
□ Pod volume mounts reference the SecretProviderClass
□ Mounted files are present inside pod at expected path


TERRAFORM RESOURCES YOU SHOULD FIND

# Company Sub
azurerm_kubernetes_cluster
azurerm_virtual_network          # company-hub-vnet
azurerm_subnet                   # aks-node-subnet
azurerm_network_security_group   # aks-nsg
azurerm_virtual_network_peering  # company-to-tunnel

# Tunnel Sub
azurerm_virtual_network          # tunnel-vnet
azurerm_subnet                   # pe-subnet (with PE network policies)
azurerm_network_security_group   # tunnel-pe-nsg
azurerm_private_endpoint         # pe-pulsar-streamnative
azurerm_private_dns_zone         # yourdomain.com
azurerm_private_dns_a_record     # pulsar → PE IP
azurerm_private_dns_zone_virtual_network_link  # to tunnel-vnet
azurerm_private_dns_zone_virtual_network_link  # to company-vnet ← critical
azurerm_key_vault                # kv-tunnel-prod
azurerm_key_vault_secret         # company-ca-trust-chain
azurerm_key_vault_certificate    # pulsar-tls-cert
azurerm_virtual_network_peering  # tunnel-to-company

# Stream Native Sub
azurerm_private_link_service     # pls-pulsar
azurerm_kubernetes_cluster       # streamnative-aks
azurerm_lb                       # stream-native-ilb
azurerm_lb_backend_address_pool
azurerm_lb_rule                  # port 6651


QUESTIONS TO ASK STREAM NATIVE TEAM

1. What is the exact PLS alias/resource ID we should target in our Private Endpoint?
2. What port does Pulsar Proxy listen on — 6651 (TLS) or 6650 (plain)?
3. What TLS cert does Pulsar Proxy present — did you install our cert from KV?
4. Have you loaded our Company CA trust chain into Pulsar's broker.conf trustCertsFilePath?
5. What authentication method is configured — mTLS only, or mTLS + JWT token?
6. If JWT token: where do we get the token and what is the token TTL?
7. What Pulsar tenant/namespace/topic should our app connect to?
8. Is Pulsar Proxy running — or are we connecting directly to brokers?
9. Are there IP allowlists on the PLS NAT config that need our PE NAT IPs added?
10. Can you share Pulsar proxy logs when we do a test connection from our side?


QUESTIONS TO ASK AZURE NETWORKING TEAM

1. Is PrivateEndpointNetworkPolicies enabled on the Tunnel PE subnet?
2. Is there a Firewall or NVA in the hub that Pulsar traffic passes through?
3. Are there any UDRs on the AKS node subnet that redirect 10.20.x.x traffic?
4. Is the VNet peering configured with UseRemoteGateways on either side?
5. Are there any Azure Policy assignments blocking Private Endpoint creation in Tunnel Sub?
6. Does the Private DNS Zone have auto-registration enabled — and could that conflict?
7. Is there a DNS Private Resolver deployed — and is it in the traffic path?
8. Are there any DDoS plans or network watchers capturing this traffic?
