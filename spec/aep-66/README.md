---
aep: 66
title: "Custom Domain Certificates"
author: Joao Luna (@cloud-j-luna)
status: Draft
type: Standard
category: Core
created: 2025-05-13
updated: 2025-07-07
estimated-completion: 2026-06-30
roadmap: major
---

 ## Abstract

 This proposal introduces a mechanism for Akash Network tenant workloads to obtain SSL/TLS certificates for their configured custom domains, enabling secure HTTPS access to deployments. 

 ## Motivation

 Currently, Akash Network deployments are accessible via the default ingress subdomain (e.g., *.ingress.akash.pub).
 To enhance the security and accessbility of deployments, tenants should have the ability to use custom domains with SSL/TLS certificates without relying on third party solutions such as Cloudflare.

 ## Technical Details

 ### Certificate Management (cert-manager)
 - `cert-manager` is a Kubernetes controller used to automate the management and issuance of TLS certificates.
 - It supports Let's Encrypt and other certificate authorities.
 - On Akash, `cert-manager` runs as part of the provider infrastructure and handles certificate issuance for Gateway API resources and associated routes
 - It uses HTTP-01 challenges to validate domain ownership.
 - Users do not directly interact with cert-manager, but it powers the automatic issuance of certs based on deployment configuration, HTTPRoutes and DNS records.

### Gateway API
- Akash uses the Kubernetes Gateway API to route external HTTP(S) traffic to tenant workloads instead of the legacy Ingress API.
- Gateway resources define listeners (entry points) for HTTP and HTTPS, while HTTPRoute resources define rules for routing traffic to services.
- The Akash provider manages Gateway/HTTPRoute creation based on the deploy.yml service definitions (expose section).
- HTTPS routing is enabled by specifying ports 443 and a custom domain in the manifest, which is translated into Gateway listeners and HTTPRoutes configured for TLS termination. 
### Deployment Manifest
 - The manifest file allows defining services, ports and accepted domains.
 - To use a custom domain:
 ```
 expose:
   - port: 80
     to:
       - global: true
   - port: 443
     to:
       - global: true
     accept:
       - "www.example.com"
 ```
 - This instructs the provider to configure Gateway API resources (Gateway listeners and HTTPRoutes) and attempt TLS certificate issuance for the domain via cert-manager.

 ### DNS Configuration
 - DNS setup is critical for domain validation and traffic routing.
 - Tenants must create a CNAME record pointing to their deployment’s HTTPS endpoint exposed by the provider’s Gateway implementation.
   - Example:
   - `www.example.com` -> `deployment123.ingress.provider.akash.network`
 - DNS propagation must complete before certificate issuance via Let's Encrypt can succeed.

 ### Certificate Lifecycle
 - Certificates are automatically requested, issued, and renewed via `cert-manager` based on the Gateway's TLS configuration.
 - Tenants do not manually manage TLS certs.
 - Failure to configure DNS correctly will prevent certificate issuance and may fall back to untrusted/self-signed certs.



 ## Implementation

An implementation leveraging  cert-manager  would simplify the whole solution by configuring the Gateway with specific annotations that trigger certificate issuing and by wiring HTTPRoutes to the HTTPS listener for the desired hostnames. A TLS configuration would also need to be added to the Gateway listener, pointing to the TLS secret referenced by  certificateRefs , which will be populated with the accepted domains.
 cert-manager  watches Gateway resources across the Akash Provider cluster when Gateway API support is enabled. If it observes a Gateway with annotations related to certificate issuing, it will ensure a Certificate resource exists that matches the hostnames configured on HTTPS listeners and that the generated TLS Secret is referenced in the listener’s  certificateRefs . An example Gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
meta
  name: my-gateway
  annotations:
    cert-manager.io/cluster-issuer: nameOfClusterIssuer
spec:
  gatewayClassName: akash-gateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
    - name: https
      protocol: HTTPS
      port: 443
      hostname: example.com
      tls:
        mode: Terminate
        certificateRefs:
          - name: my-gateway-cert
```

With this, user workloads will be provided a valid and automatically managed certificate for their custom domains through the provider’s Gateway API data plane.
