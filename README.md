# IDTECH Ansible HAProxy Lab

## Architecture

This lab uses two Ubuntu virtual machines:

- control01 - Ansible control node and managed host
- app01 - remote managed host running backend and HAProxy

Current lab network:
- control01: 192.168.104.5
- app01: 192.168.104.6
- app01 hostname: lima-app01.internal

The inventory uses the resolvable hostname instead of hard-coding the remote IP.

## Ports

- Backend: 127.0.0.1:9000
- HAProxy frontend: 8080
- SSH: key-based authentication from control01 to app01

## Project Structure

Responsibilities are separated into four roles:

- common
- control_node
- backend
- haproxy

Variables are stored under group_vars and host_vars.

## Control Node

control01 is included in the Ansible inventory with local connection.

Ansible creates:

/opt/idtech-control/managed-by-ansible.txt

This proves that the control node is also managed by the playbook.

## Backend

A minimal Python HTTP backend is deployed on app01.

Main endpoint returns:

IDTECH backend service | host=app01

Health endpoint:

/health

The backend runs under a dedicated system user and is managed by systemd.

Backend port:

127.0.0.1:9000

## HAProxy

HAProxy runs on app01 and listens on port 8080.

It forwards requests to:

127.0.0.1:9000

The HAProxy configuration is generated from a Jinja2 template.

Before activation, the candidate configuration is validated using:

haproxy -c

A handler reloads HAProxy only when the active configuration changes.

## Failure Protection

HAProxy configuration deployment uses Ansible block/rescue.

An invalid configuration was intentionally introduced.

The validation step rejected the invalid candidate configuration and preserved the existing working configuration.

HAProxy continued serving requests after the failed deployment.

## Ansible Features Used

- inventory
- variables
- group_vars
- host_vars
- facts
- conditions
- loops
- register
- handlers
- Jinja2 templates
- block/rescue
- systemd/service modules
- uri health checks

## Execution

Check connectivity:

ansible all -m ping

Check syntax:

ansible-playbook site.yml --syntax-check

Run playbook:

ansible-playbook site.yml

## Validation

Direct backend test:

curl http://127.0.0.1:9000/

HAProxy test:

curl http://127.0.0.1:8080/

Expected response:

IDTECH backend service | host=app01

## Idempotency

The playbook was executed a second time without infrastructure changes.

The second normal execution completed with changed=0.

This demonstrates that the Ansible solution is idempotent.

## Security

Private SSH keys, passwords and secrets are not stored in this repository.

## Command Module Justification

The `ansible.builtin.command` module is used only for HAProxy syntax validation:

`/usr/sbin/haproxy -c -f /tmp/haproxy.cfg.candidate`

This is intentional because HAProxy provides its own native configuration syntax checker through the `-c` option. The task uses `changed_when: false` because this validation command does not modify system state.
