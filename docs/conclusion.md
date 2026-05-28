# Conclusion

## What You Accomplished

Over the five labs in this session you built a complete, multi-domain automation pipeline using Cisco's Net-as-Code methodology:

| Lab | Domain | What You Built |
|---|---|---|
| Lab 1 | ACI | Tenants, VRFs, BDs, EPGs, contracts, L3Outs, access policies |
| Lab 2 | SD-WAN | VPN and interface feature templates with SGT propagation enabled |
| Lab 3 | Catalyst Center | Two-fabric SDA hierarchy with Campus and IoT VNs and anycast gateways |
| Lab 4 | ISE | TrustSec SGTs, SGACLs, policy matrix, and network device registration |
| Lab 5 | Multi-Domain | BGP transit, L3 handoffs (CATC borders ↔ SD-WAN edges), device templates |
| Lab 6 | End-to-End Testing | End-to-end testing of the multi-domain integration |

Every configuration object was defined in a YAML data file, validated against a JSON Schema, planned with `terraform plan`, and deployed with `terraform apply` — reproducibly and without manual GUI interaction.

## Key Takeaways

**Data / logic separation scales.** By keeping your intent in YAML and the execution logic in a versioned Terraform module, you can manage hundreds of objects by editing simple text files. Domain expertise is encoded once in the module; operators use it without learning provider internals.

**A consistent pipeline across domains reduces risk.** Every domain — ACI, SD-WAN, Catalyst Center, ISE — goes through the same lint → validate → plan → deploy stages. The schema validation stage catches errors before they reach the controller, and `terraform plan` gives you a change preview before any state is modified.

**Cross-domain policy is a first-class object.** TrustSec SGTs defined in ISE are referenced by name in Catalyst Center (`security_group_name`), propagated by SD-WAN (`propagate_sgt: true`), and enforced at the fabric edge. A single YAML edit in ISE ripples through all three enforcement points via the pipeline.

**Declarative state is self-documenting.** The `data/` directory in each repository is a living specification of what the network should look like. It is version-controlled, reviewable via pull request, and auditable via `git log`.

## Next Steps

To continue developing your Net-as-Code skills:

- **Explore the full data model:** Each module documents every supported YAML key at [netascode.cisco.com](https://netascode.cisco.com/docs/data_models/overview/)
- **Add schema validation to your own data:** Run `pytest -m validate` against any data directory to test YAML against JSON Schema
- **Integrate with your Git workflow:** The GitLab CI pipeline examples in this lab are starting points for your own pipeline
- **Expand the policy matrix:** Add new SGTs and matrix entries in ISE and watch the pipeline push them to all enforcement points

## Related Sessions at Cisco Live

- [LTRENS-3751](https://www.ciscolive.com/emea/learn/session-catalog.html?search=LTRENS-3751) — Automating SD-Access Fabric Deployment with Cisco Catalyst Center, ISE, and Terraform
- [LTRATO-2223](https://www.ciscolive.com/emea/learn/session-catalog.html?search=LTRATO-2223) — FastForward SD-WAN Deployment and Management with SD-WAN as Code
- [LTRDCN-2459](https://www.ciscolive.com/emea/learn/session-catalog.html?search=LTRDCN-2459) — Cross-Technology Automation of ACI and FMC with Network-as-Code from Day 0 to Day 2
- [TECSEC-2834](https://www.ciscolive.com/emea/learn/session-catalog.html?search=TECSEC-2834) — Optimizing Cisco Security: Automate Provisioning and Configuration with Ansible and Terraform
- [BRKENT-2115](https://www.ciscolive.com/emea/learn/session-catalog.html?search=BRKENT-2115) — Automate Your Catalyst SD-WAN with Network as Code
- [LTRDCN-2678](https://www.ciscolive.com/emea/learn/session-catalog.html?search=LTRDCN-2678) — Become an ACI Automation Ninja in Just Four Hours

## Additional Resources

| Resource | URL |
|---|---|
| Net-as-Code Documentation | https://netascode.cisco.com/docs/overview/ |
| Data Model Reference | https://netascode.cisco.com/docs/data_models/overview/ |
| CI/CD Pipeline Documentation | https://netascode.cisco.com/docs/cicd_pipeline/overview/ |
| Terraform ACI Provider | https://registry.terraform.io/providers/CiscoDevNet/aci |
| Terraform SD-WAN Provider | https://registry.terraform.io/providers/CiscoDevNet/sdwan |
| Terraform Catalyst Center Provider | https://registry.terraform.io/providers/CiscoDevNet/catalystcenter |
| Terraform ISE Provider | https://registry.terraform.io/providers/CiscoDevNet/ise |
