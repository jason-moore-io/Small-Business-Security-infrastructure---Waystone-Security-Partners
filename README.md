# Waystone Security Partners

**Small business security infrastructure — internal reference**

A role-per-machine infrastructure plan for a nine-person security consultancy, built entirely from repurposed hardware: six compute nodes, a Wi-Fi access point, a 20 TB mirrored DAS, and a managed switch, arranged into eight trust zones so a client engagement gone wrong can never reach the firm's own backups, and client files stay walled off from everything else.

> Waystone Security Partners is a nine-person consultancy used here for illustration: security assessments, an incident-response retainer, and light managed monitoring for small and mid-sized clients. The hardware below is the same fleet a scrappy two-year-old firm would actually have lying around — old laptops, a mini PC, a leftover Pi — pressed into a real production architecture instead of a rack of new appliances.

| | |
|---|---|
| **6** | repurposed compute nodes running the firm's whole stack |
| **10 TB** | usable after RAID 1, for client file retention |
| **8** | VLANs separating client data from everything else |
| **1** | chokepoint every packet crosses: the office firewall |

---

## 01 — Infrastructure manifest

What each machine does for the firm, and nothing else — a nine-person shop can't afford the staff time to reason about boxes with five overlapping jobs, and neither can its cyber-insurance questionnaire.

| Device | Specs | Assigned role | VLAN |
|---|---|---|---|
| ASUS VivoMini | 16 GB RAM · 1 TB internal | Client file server, RAID storage, and the firm's continuity knowledge base | 30 · Storage |
| WD My Book Duo (20 TB) | 2-bay DAS, USB/Thunderbolt | RAID 1 mirror attached to the VivoMini — not independently networked | Part of 30 |
| MacBook Pro 13" (2012) | 16 GB RAM · 500 GB + 1 TB secondary | Backup & disaster-recovery node only, for client engagement records and firm data. Nothing else runs here | 40 · Backup |
| MacBook Air 11" (2012) | 4–8 GB RAM · small SSD | Internal log monitoring and the case-tracking database, sized to what this hardware can actually run | 50 · Monitoring |
| Omarchy Linux (Lenovo Yoga) | 16 GB RAM · 500 GB · 2019 | Security testing workstation — where staff run authorized client assessments and validate tooling before it ever touches a client network | 25 · Sec. Testing |
| Beelink PUC mini PC | 8 GB RAM · 500 GB | Firewall, router, and VPN gateway — the only box with a foot in every VLAN | Edge · all VLANs (trunk) |
| Wi-Fi router (AP mode) | wireless bridge · no routing/DHCP | Staff Wi-Fi — flashed to AP/bridge mode rather than acting as its own router | 15 · Wireless |
| Raspberry Pi 1 | ARMv6 · 256–512 MB RAM | Internal test range for validating tooling and findings before staff run them against a live client environment | 60 · Range |

---

## 02 — Network topology

One physical trunk leaves the firewall; the switch fans it out into eight tagged VLANs. Trust decreases left to right — from the zone the owner administers from, to the zone assumed hostile.

```
                              WAN
                               │
                               ▼
   Tailscale mesh  ◄──VPN──  Beelink PUC · OPNsense
  (consultants on-site,        firewall / router
   optional cloud VM)          WireGuard + Tailscale gateway
                               │
                               │  trunk · every VLAN tagged
                               ▼
                        Managed office switch
                               │
        ┌───────────┬─────────┼─────────┬───────────┬───────────┬───────────┐
        │           │         │         │           │           │           │
     VLAN 10     VLAN 20   VLAN 15   VLAN 25     VLAN 30     VLAN 50     VLAN 40      VLAN 60
   Management    Trusted  Wireless  Sec.Testing  Storage    Monitoring   Backup        Range
   owner/admin   staff    Wi-Fi     Omarchy      VivoMini   MacBook Air  MacBook Pro   Raspberry Pi 1
   + VPN         laptops  router,   Linux, LYoga + WD Duo   Loki +       (2012),       + test targets
   landing                AP mode   16GB, client RAID 1,    Postgres,    Restic/Borg   no route
                                    engagement   continuity one-way log  repo, pulls   anywhere —
                                    tooling      knowledge  intake       only,         tooling
                                                 base                    accepts       validation
                                                                         nothing       only
                                                                         inbound         │
                                                                            │            │
                                                                            ▼            │
                                                                      Backblaze B2  ◄─────┘
                                                                    (offsite, 3rd leg
                                                                       of 3-2-1)
```

Every VLAN reaches the firewall and nothing else by default. The backup node has no inbound arrow at all — it reaches out to Storage and to Backblaze; nothing reaches in. Wireless mirrors Trusted's access but stays its own segment since RF crosses walls; the security testing workstation is the only zone besides Management allowed to reach into Range.

**Legend:** Trusted zones (10 / 15 / 20 / 30) · Security testing workstation (25) · Monitoring (50) · Backup, most restricted (40) · Range, assumed hostile (60)

---

## 03 — VLAN rules of engagement

| Zone | Members | Inbound allowed | Outbound allowed |
|---|---|---|---|
| VLAN 10 Management | Switch & firewall mgmt interfaces, owner's VPN landing zone | VPN clients only | Everything (admin only) |
| VLAN 20 Trusted | Staff laptops at their desks | From Management | Storage (client files, knowledge base), Monitoring (dashboards), internet |
| VLAN 15 Wireless | Office Wi-Fi router (AP/bridge mode) and staff wireless devices | From Management only, for setup | Storage (files, knowledge base), Monitoring (dashboards), internet |
| VLAN 25 Sec. Testing | Omarchy Linux workstation (2019 Lenovo Yoga) | From Management only, for setup | Storage (knowledge base, file shares), Range (active validation only), Monitoring (push logs/results), filtered internet egress |
| VLAN 30 Storage | ASUS VivoMini + WD My Book Duo — client files and the knowledge base | SMB/NFS/HTTP from Trusted, Wireless, Management, and Security Testing; a scoped read-only pull account from Backup | Internet only for scheduled knowledge-base content updates |
| VLAN 50 Monitoring | MacBook Air (Loki, Promtail, Postgres, Grafana, case tracking) | Log-forward push from every zone, one port, one direction | None — it never initiates a connection out |
| VLAN 40 Backup | MacBook Pro (2012), Restic/Borg repository of client engagement records | None. Zero listening services reachable from any other VLAN | Read-only pull from Storage; push to Backblaze B2 only |
| VLAN 60 Range | Raspberry Pi 1, any other disposable validation targets | From Management (setup/reset) and from Security Testing (active validation only) | None by default; enabled per-validation run and switched back off |

Confirm the office switch does real 802.1Q tagged VLANs, not just port-based isolation, before wiring this up. If it turns out to be port-based only, give the firewall a second NIC (a USB3-to-gigabit adapter works fine) and split zones across physical ports instead.

---

## 04 — Bringing office Wi-Fi into the VLAN scheme

A consumer Wi-Fi "router" is a router, DHCP server, and NAT gateway all in one, and left as-is it'll happily create its own invisible subnet the firewall can't see into — every staff device on Wi-Fi would become a single blur behind the router's one WAN IP, and every VLAN rule in Section 03 would stop meaning anything for whoever's on wireless. Flash it to AP (bridge) mode first, which turns off its own DHCP/NAT/routing and leaves it as a dumb wireless-to-Ethernet bridge, then land its LAN port on a switch port assigned untagged to VLAN 15 so the firewall hands out DHCP the same way it does for every other zone.

If the router supports multiple SSIDs mapped to different VLAN tags, that's the natural way to add a walled-off guest network later for clients visiting the office — without touching anything else here. WPA3, a passphrase used nowhere else, and WPS turned off are the baseline; RF carries past the office walls in a way a switch port never does, so encryption is doing the job physical access does everywhere else in this design.

---

## 05 — The security testing workstation

The Omarchy Linux machine (Hyprland-based Arch, 2019 Lenovo Yoga, 16 GB RAM) is where staff run the assessments the firm sells, and where new tooling gets tried out before it ever touches a client's network. That second use is the real reason it sits on its own VLAN rather than in Trusted with everyone's laptops: standard practice before running anything against a paying client is to validate it against something disposable first, and this is that something. If a proof-of-concept or a downloaded tool turns out to misbehave, that stays contained to VLAN 25 instead of landing next to the machines running payroll and email.

Being Arch-based, Omarchy doesn't ship with an assessment toolkit preinstalled, but nearly everything a security engineer needs is one `pacman` or AUR install away — nmap, Metasploit, Burp Suite, Wireshark. The 16 GB of RAM comfortably runs a disposable VM or two locally (QEMU/KVM) for testing without touching the Range hosts directly.

Give it outbound access to Storage for the knowledge base and file shares, to Monitoring so findings get logged, and to Range only while a validation run is deliberately switched on. No path from Trusted into it — staff laptops used for email and invoicing have no reason to reach a box running assessment tooling, and it has no reason to reach them back.

---

## 06 — Client storage and RAID

The WD My Book Duo's own controller can do RAID 0, RAID 1, or JBOD across its two bays. RAID 1 is the right call for a firm holding client engagement data: 10 TB usable instead of 20 TB raw, but a single drive failure costs nothing. Put it behind the VivoMini running TrueNAS SCALE rather than the WD utility's own mirroring — TrueNAS adds ZFS checksums (silent corruption gets caught, not just drive death), snapshots, and a Docker/app layer that can run the knowledge-base stack on the same box that holds the redundant storage.

That pairing is deliberate: assessment reports, evidence packages, and client deliverables belong on the mirrored pool rather than a single internal SSD, and so does the reference material staff need mid-engagement. RAID isn't a backup, though — it protects against a dead drive, not an accidental deletion or a ransomware incident at the firm itself, which is what the separate backup node in VLAN 40 is for.

---

## 07 — Backup, kept genuinely separate from client data

The control that matters most here: the backup node holds credentials the storage node never sees, and it's always the one to initiate the connection. Give the MacBook Pro a restricted, read-only account on the storage node (rsync-over-SSH with a forced command, or a read-only NFS export scoped to the client datasets) and have it pull nightly. It writes into a local Restic or Borg repository on its own secondary 1 TB drive — encrypted, versioned, and append-only if Restic's `--append-only` mode is enabled on the repo. From there it pushes a second, independently-encrypted copy to Backblaze B2, giving the firm the classic 3-2-1: the live RAID pool, the local backup copy, and an offsite one.

If the storage node is ever compromised, there's nothing on it that can reach into VLAN 40, delete the backup repository, or read the credentials that write to B2. For a firm whose engagement contracts likely commit to retaining client records for years, that isolation is doing real compliance work, not just tidiness.

---

## 08 — Where the continuity knowledge base actually needs to live

The knowledge base runs on Project NOMAD (Crosstalk Solutions' offline knowledge and AI server — Kiwix/Wikipedia, Kolibri courses, ProtoMaps, optional Ollama, CyberChef, FlatNotes), which is Docker-based and wants no direct internet exposure once it's running. The rationale for a security firm specifically: an incident-response engagement is, by definition, often the moment a client's own network and internet access are unreliable. Staff on-site need documentation, playbooks, and reference material that don't depend on the very network that's having a bad day. The VivoMini's 16 GB runs the knowledge services comfortably alongside file serving as long as any local LLM stays small — a 3B-class quantized model through Ollama — or local AI gets skipped in favor of a hosted API when there is internet to spare.

Keep it on VLAN 30 with the storage it's built from, reachable from Trusted, Wireless, and Security Testing for reference during an engagement, and let it update its content only on a schedule the firm controls.

---

## 09 — Watching the firm's own network, not just clients' networks

A full Elasticsearch/Kibana stack wants more RAM than the MacBook Air can spare. Grafana Loki plus Promtail does the same job at a fraction of the footprint, and a small PostgreSQL instance alongside it backs the firm's own case-tracking — engagement status, findings, hours — instead of a dedicated SaaS tool the firm would otherwise pay for. Every other host, including the security testing workstation, runs a Promtail agent configured to push only, into VLAN 50, on one port; the monitoring node itself never opens a connection back out. There's an obvious credibility angle too: a firm that sells security assessments should be watching its own network at least as closely as it watches a client's.

---

## 10 — Remote access for consultants, and the cloud tie-in

Tailscale on the Beelink gives consultants working on-site at a client's office a way to reach the firm's file server and knowledge base without ever joining the client's network with anything that also has a path back to the office — a real professional-hygiene requirement, not a convenience. Its ACL tags map directly onto the VLAN trust tiers above: a laptop tagged `admin` reaches Management and Trusted, nothing else can touch Backup or Range. A cheap $5–6/month VPS joined to the same mesh is a stable relay for the rare double-NAT client site, and Backblaze B2 handles the offsite leg of the backup plan from Section 07 — S3-compatible, natively supported by Restic, and priced well below the equivalent AWS storage class for a firm watching its margins.

**Running it on OPNsense:** install the community `os-tailscale` plugin from Firmware → Plugins rather than compiling anything by hand. Advertise a route for every VLAN, but approve each one individually in the Tailscale admin console — there's no reason to expose Storage, Backup, or Range to a traveling laptop until the firm actually needs it that deep. Three things to expect: OPNsense drops Tailscale traffic by default until an explicit pass rule is added on the new interface; its NAT type pushes most connections through Tailscale's DERP relays rather than a direct link, which is fine for admin traffic but not something to expect gigabit throughput from; and its DNS rebind protection can silently reject internal hostnames resolved over the tailnet unless the firm's domain is exempted.

---

## 11 — Hardening checklist

- **Confirm tagged VLAN support** on the actual switch model before wiring the trunk; port-based-only switches need the two-NIC firewall workaround above.
- **Default-deny between every VLAN** at the firewall, with explicit allow rules only for the flows listed in Section 03.
- **Disable unused switch ports** and, if the model supports it, enable port security or 802.1X so a visitor can't just plug in and hop VLANs.
- **Move the 2012 Macs and the VivoMini off stock macOS** onto a current Linux server distro. Old macOS releases stop getting security patches; a firm selling security work can't be running unpatched infrastructure of its own.
- **Treat the Raspberry Pi 1 as inherently unpatchable** and let VLAN 60's total isolation be the mitigation, rather than trying to harden the Pi itself.
- **Confirm the Wi-Fi router can run in AP/bridge mode** before wiring it in, and pair it with WPA3 and a passphrase used nowhere else. Add a separate guest SSID/VLAN before any client visits and asks for Wi-Fi.
- **Keep the security testing workstation's egress filtered and logged** — it's the one host expected to run untrusted or experimental code, and the firm's own insurer will likely ask about exactly this.
- **Enforce the pull-only, credential-asymmetric backup pattern** from Section 07 — the single highest-value control against ransomware reaching client records.
- **Enable MFA** on the OPNsense admin login and the Tailscale account gating remote access.
- **Encrypt at rest and in transit everywhere it matters**: Restic/Borg's native encryption for backups, TLS to Backblaze, and a passphrase-protected ZFS pool.
- **Keep this document current and ready to hand over.** Client contracts, cyber-insurance renewals, and SOC 2-adjacent questionnaires will ask how client data is segmented and protected; this architecture is most of that answer, provided it still matches what's actually running.

---

## 12 — Suggested build order

1. **Firewall and VLANs first** — Get OPNsense running on the Beelink and all eight VLANs defined on the switch before anything else joins the network.
2. **Storage node** — Rack the VivoMini with the WD Duo, build the RAID 1 pool in TrueNAS SCALE, and get client file sharing working on VLAN 30 before adding anything else to that box.
3. **Backup node, before there's client data to lose** — Stand up the MacBook Pro on VLAN 40 with its restricted pull account and first Restic snapshot while the storage node's data set is still small — verify a full restore once, now.
4. **Knowledge base on the storage node** — Install the NOMAD-based stack on the VivoMini, point its data volumes at the RAID pool, and decide whether local Ollama models are worth the shared RAM.
5. **Monitoring node** — Bring up Loki/Promtail/Postgres on the MacBook Air and point every other host's Promtail agent at it before relying on any of this for detection.
6. **Wireless access** — Flash the Wi-Fi router to AP mode, land it on VLAN 15, and confirm a staff device actually gets a lease from OPNsense before trusting it for daily use.
7. **Security testing workstation** — Join the Omarchy Yoga to VLAN 25 and confirm it can reach Storage and Monitoring but not Trusted or Backup, before pointing it at anything real.
8. **Range, last and disposable** — Only bring the Raspberry Pi (and any other validation targets) online once the other zones and their firewall rules are proven, so the one zone built to absorb mistakes has nothing behind it to reach except the workstation that just isolated it.

---

*Waystone Security Partners is a fictitious company, used here to illustrate how a small consultancy could run its entire internal security stack on the same repurposed-hardware architecture: a CTF-and-general-learning home lab restyled around client files, engagement records, and staff access instead of personal projects. Revisit the VLAN table if the switch turns out to be port-based rather than 802.1Q tagged.*
