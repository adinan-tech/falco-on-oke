# Falco on OKE

## Overview

This stack deploys an Oracle Kubernetes Engine (OKE) cluster with a bastion and an operator VM to demonstrate how Falco detects and reports runtime threats on Kubernetes.

Falco, Falcosidekick, and Falcosidekick-UI are installed automatically through Helm.  
The Falcosidekick UI allows you to visualize Falco alerts in real time.

## Getting Started

### Prerequisites

- An OCI account with permissions for Container Engine and Compute resources.  
  See the required IAM policies [here](https://docs.oracle.com/en-us/iaas/Content/ContEng/Concepts/contengpolicyconfig.htm#policyforgroupsrequired).
- Access to OCI Resource Manager (ORM) to create and apply stacks.  
- SSH access from your local machine (for port forwarding).

### Deploy via OCI Resource Manager

1. Clone this repository.  
2. Create a new stack in OCI Resource Manager.  
3. Upload the folder **oci_oke_falco**.  
4. Configure the required variables (compartment, shape, image, CIDRs).  
5. Apply the stack.  
6. When the job completes, note the output section, which provides:
   - Bastion Public IP
   - Operator Private IP
   - SSH to Operator command (for direct access)

## Verify Falco Deployment

1. SSH into the operator VM using the ORM output:

   ```bash
   ssh -J opc@<bastion_public_ip> opc@<operator_private_ip>
   ```

2. Check Falco components:

   ```bash
   kubectl get pods -n falco
   ```

   You should see output similar to:

   ```
   pod/falco-...                              2/2   Running
   pod/falco-falcosidekick-...                1/1   Running
   pod/falco-falcosidekick-ui-...             1/1   Running
   pod/falco-falcosidekick-ui-redis-0         1/1   Running
   ```

**Interpretation:**

- **falco-...** → Main Falco DaemonSet (sensors + driver loader).  
- **falco-falcosidekick-...** → Forwards Falco alerts to the Web UI and other sinks.  
- **falco-falcosidekick-ui-...** → Web dashboard to visualize alerts.  
- **falco-falcosidekick-ui-redis-0** → Redis backend storing recent events.

## Access Falcosidekick UI

You will use two SSH tunnels: one from your local machine through the bastion to the operator, and one Kubernetes port-forward inside the cluster.

1. On the operator, forward the Falco UI service:

   ```bash
   kubectl -n falco port-forward svc/falco-falcosidekick-ui 2802:2802
   ```

   Leave this terminal open.

2. On your local machine, run:

   ```bash
   ssh -L 2802:localhost:2802 -o ProxyCommand="ssh -W %h:%p opc@<bastion_public_ip>" opc@<operator_private_ip>
   ```

   Leave this running to keep the tunnel active.

3. Open the Falcosidekick UI in your browser:

   ```
   http://127.0.0.1:2802
   ```

4. Login with:

   ```
   admin / admin
   ```

## Validate Falco Installation

1. Confirm Falco and Sidekick are running:

   ```bash
   kubectl -n falco logs ds/falco -c falco --tail=20
   kubectl -n falco logs deploy/falco-falcosidekick --tail=20
   ```

2. You should see output similar to:

   ```
   [INFO] : Falcosidekick version: 2.32.0
   [INFO] : Enabled Outputs: [WebUI]
   [INFO] : Falcosidekick is up and listening on :2801
   ```

3. If all components are ready, Falco is now monitoring container and host activity.

## Cleanup

When finished, destroy the stack from ORM.

## Notes

- Falco outputs events to Falcosidekick on port 2801.  
- The Web UI is exposed internally on port 2802.  
- The Helm deployment installs Falco, Falcosidekick, and Falcosidekick-UI with default settings.  
