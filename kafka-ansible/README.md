
# Kafka Ansible Deployment

This repository contains an Ansible playbook for deploying a Kafka cluster with Zookeeper on multiple nodes. The playbook automates the installation, configuration, and management of Kafka and Zookeeper services.

## Prerequisites

1. **Ansible**: Ensure Ansible is installed on your control machine.
2. **Target Machines**: Prepare the target machines with the following:
   - SSH access enabled.
   - Sudo privileges for the Ansible user.
   
3. **Inventory Configuration**: Update the `inventory` file with the IP addresses and credentials of your target machines.

## Repository Structure

- `kafka.yml`: Main playbook for deploying Kafka and Zookeeper.
- `inventory`: Defines the target hosts and their variables.
- `group_vars/`: Contains group-level variables.
- `host_vars/`: Contains host-specific variables.
- `logs/`: Directory for Ansible logs.
- `ansible.cfg`: Ansible configuration file.

## Usage

1. **Clone the Repository**:
   ```bash
   git clone <repository-url>
   cd kafka-ansible
   ```

2. **Update Variables**:
   - Modify `group_vars/kafka.yml` to set Kafka and Zookeeper versions, and node IPs.
   - Update `host_vars/` files for host-specific configurations.

3. **Run the Playbook**:
   Execute the playbook to deploy the Kafka cluster:
   ```bash
   ansible-playbook -i inventory kafka.yml
   ```

4. **Verify Deployment**:
   - Check the status of Zookeeper and Kafka services on each node:
     ```bash
     systemctl status zookeeper
     systemctl status kafka
     ```

5. **Logs**:
   - Deployment logs are stored in `logs/ansible-kafka.log`.

## Notes

- Ensure the `ansible_user` specified in the inventory has the necessary permissions to install software and manage services.


## Troubleshooting

- If the playbook fails, check the logs in `logs/ansible-kafka.log` for detailed error messages.
- Verify network connectivity between the control machine and target nodes.
- Ensure the specified Kafka and Zookeeper versions are available for download.


