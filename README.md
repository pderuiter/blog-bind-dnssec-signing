# blog-bind-dnssec-signing

Companion code for the blog post
[Signing Your DNS Zones with DNSSEC on BIND Authoritative Servers](https://blog.bsdserver.nl/signing-your-dns-zones-with-dnssec-on-bind-authoritative-servers/).

It builds the post's "Complete Configuration Example": a BIND authoritative server that signs its
zones automatically with `dnssec-policy` and inline signing.

- `dnssec-policy "production"`: ECDSAP256SHA256 KSK (unlimited lifetime) and ZSK (90 days),
  14-day signatures re-signed 5 days before expiry, NSEC3 with 0 iterations and no salt (RFC 9276)
- `example.com` and `1.168.192.in-addr.arpa` as primary zones with `inline-signing yes`
  (your unsigned zone file stays untouched, BIND keeps the `.signed` copy)
- `key-directory` and `managed-keys-directory` created under `/var/cache/bind`
- zone transfers only with the TSIG key `transfer-key` (or none when no secret is set), `also-notify` to your secondaries
- a `dnssec` log channel in `/var/log/named/dnssec.log` and `querylog yes`
- `/usr/local/bin/check-dnssec.sh` to warn before signatures expire

BIND creates the keys itself on first start. Publishing the DS record at your registrar and KSK rollovers
stay manual steps, as described in the post.

## Requirements

- Terraform 1.6+ and a Proxmox VE cluster with an API token (only if you want Terraform to create the VM)
- Ansible core 2.16+ on your workstation
- A target running Ubuntu 24.04 (tested; the post uses the Debian layout `/etc/bind`, `/var/cache/bind`)

## Quick start

```bash
# 1. Create the VM (optional, skip if you already have a server)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Configure BIND
cd ../ansible
ansible-galaxy collection install -r requirements.yml
cp group_vars/all/vault.yml.example group_vars/all/vault.yml   # set the TSIG secret
ansible-vault encrypt group_vars/all/vault.yml
ansible-playbook site.yml --ask-vault-pass

# 3. Run the post's checks (DNSKEY/RRSIG, NSEC3PARAM, rndc dnssec -status, dnssec-verify, delv, expiry script)
ansible-playbook ../tests/verify.yml
```

Without Terraform, copy `ansible/inventory.example.yml` to `ansible/inventory.yml` and edit it.

After the first run, get the DS record for your registrar with:

```bash
dnssec-dsfromkey /var/cache/bind/keys/Kexample.com.+013+<ksk-id>.key
```

## Variables

| Variable | Default | Description |
| --- | --- | --- |
| `bind_dnssec_signing_zones` | example.com and 1.168.192.in-addr.arpa from the post | Zones with name, file, policy, serial, SOA and records |
| `bind_dnssec_signing_policies` | `production` from the post | `dnssec-policy` statements (keys and options) |
| `bind_dnssec_signing_transfer_key_secret` | `""` | TSIG secret for `transfer-key`; empty means `allow-transfer { none; }` |
| `bind_dnssec_signing_also_notify` | `[]` (`192.168.1.2` in group_vars) | Secondaries to notify |
| `bind_dnssec_signing_key_dir` | `/var/cache/bind/keys` | `key-directory` |
| `bind_dnssec_signing_managed_keys_dir` | `/var/cache/bind/managed-keys` | `managed-keys-directory` |
| `bind_dnssec_signing_zone_dir` | `/var/cache/bind/zones` | Zone files |
| `bind_dnssec_signing_check_zone` | `example.com` | Zone checked by `check-dnssec.sh` |
| `bind_install_source` | `distro` | `distro` (BIND 9.18 on Ubuntu 24.04) or `isc` (ppa:isc/bind, BIND 9.20) |

Secrets go in `group_vars/all/vault.yml` (see `vault.yml.example`), encrypted with ansible-vault.

## Layout

- `terraform/` creates Ubuntu 24.04 VMs on Proxmox VE (bpg/proxmox provider) and outputs an Ansible inventory
- `ansible/roles/bind_install` installs BIND
- `ansible/roles/bind_dnssec_signing` writes the policies, zones and the check script
- `tests/verify.yml` runs the functional checks from the post

## License

MIT
