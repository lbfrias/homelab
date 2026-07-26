# 01 — Ansible inventory + SSH access

**What to build:** A minimal Ansible setup that can reach all 3 nodes. Run a ping playbook and get success on peggy, yelena, and xialing.

**Blocked by:** None — can start immediately

**Status:** ready-for-agent

- [ ] Create `ansible/inventory/hosts.yml` with all 3 nodes (peggy, yelena, xialing)
- [ ] Group nodes: `rpi` (peggy, yelena) and `x86` (xialing)
- [ ] Set connection vars (ansible_user, ansible_ssh_private_key_file)
- [ ] Create `ansible/playbooks/ping.yml` that runs `ansible.builtin.ping`
- [ ] Verify: `ansible-playbook -i inventory/hosts.yml playbooks/ping.yml` succeeds on all nodes
