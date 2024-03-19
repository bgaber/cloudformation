# How To Modify AWX For New VPC

1. Create new VPC

- It is important to have NAT Gateway so that the EC2 instances can reach Internet for updates.
- If using VPC Peering then deploy the `cloud-transformation-vpc-without-vgw-with-peering.yaml` Cloud Formation template from the BitBucket cloudformation/Canada Cloud Transformation repository.
- If using VPC Peering then, in both VPC peers, ensure route exists from private subnets to VPC Peering Connection for peered VPC CIDR.
- If not using VPC Peering, then deploy the `cloud-transformation-vpc-without-vgw.yaml` Cloud Formation template from the BitBucket cloudformation/Canada Cloud Transformation repository.
- If not using VPC Peering, then, in both VPCs, ensure route exists from private subnets to VPN for peered VPC CIDR.

2. Copy some Security Groups from existing VPC to the new VPC.

- typically for Linux you will want three SGs: Standard, Cisco AnyConnect and Redhat Capsule and for Windows you will want two SGs: Standard and Cisco AnyConnect
- you can easily determine SGs to copy using Ansible code from existing VPC, e.g. `caprod.yml`

```yaml
 - name: Setup Variables for CA Prod
  set_fact:
    awsarnrole: "arn:aws:iam::172507017890:role/sre-ec2-role-assumed"
    target_region: "ca-central-1"
    # Linux list of Standard, Cisco AnyConnect and Redhat Capsule Security Groups
    linux_sec_gps: ["sg-1a06b672", "sg-047b211de3df58c62", "sg-03db9d3822cdc6489"]
    # Windows list of Standard and Cisco AnyConnect Security Groups
    windows_sec_gps: ["sg-1a06b672", "sg-047b211de3df58c62"]
    # Key pair to use on the instance.
    target_key: "CANADA PROD"

- name: Setup Subnetting Variables CA Prod A
  set_fact:
    target_subnet_id: "subnet-dba098b2"
  when: (netside == "A")

- name: Setup Subnetting Variables CA Prod B
  set_fact:
    target_subnet_id: "subnet-fef0a185"
  when: (netside == "B")
```

3. Modify code in the Bitbucket `infra-utility-ec2provision-20` repository

- ensure you have a local forked copy of repository and use proper git Push and Pull Requests to update the code
- follow example in [this document to fork a repository](https://www.atlassian.com/git/tutorials/making-a-pull-request#example)
- DO NOT update the code directly on the BitBucket repository.
- create new VPC playbook modelled on Ansible code from existing VPC, e.g. `cacamh.yaml`
- probably will require Security Group Ids from step 2 and Subnet Ids from the VPC creation in step 1.
- modify `main.yaml` to include the new VPC playbook, for example:

```yaml
    - import_tasks: catransformationtest.yml
      when: (buildenv == "ca-transformation-test")
```

4. Modify AWX Server Build Template, by click on name, then on the Survey tab and then on the <span style="color:blue">Which environment are we deploying?</span><span style="color:red">*</span> link.

![Alt text](images/awx-survey.png?raw=true "AWX Server Build Template Survey")

![Alt text](images/awx-edit-env.png?raw=true "Server Build Template Edit Environment")

5. Enable routing between new VPC and US Shared VPC where AWX server (SPLAWSSREAWX01) resides.

- how this step is accomplished depends on whether VPC routing is used or routing through CompuCom network is used.

6. In the Security Group that protects the Windows Domain Controllers open LDAP Port 389 and LDAPS Port 636 from the new VPC’s CIDR.

- at the time this was written the Windows DCs were 10.251.2.179, 10.251.2.156 and 10.251.5.163 and the image below shows the modification to `sg-040786b1cfc1c6e4f` for LDAP and LDAPS that protects spwawswdc01 (10.251.2.179), SPWAWSINFDC01 (10.251.2.156) and SPWAWSINFDC02 (10.251.5.163).

## Problems and their fixes ##

### 1. Privilege Escalation Error ###
- In the `infra-utility-ec2provision` repository the following `Configure New Linux Instance` Play in the `main.yml` file results in a Privileged Escalation error:

```yaml
- name: Configure New Linux Instance
  hosts: new_launch_linux
  gather_facts: true
  become: true
```

- In the AWX deployment log this error is seen:

```bash
216  PLAY [Configure New Linux Instance] ********************************************
217
218  TASK [Gathering Facts] *********************************************************
219  fatal: [STLACATVMARFE1]: FAILED! => {"msg": "Timeout (62s) waiting for privilege escalation prompt: "}
```

- this problem is fixed by permitting LDAP and LDAPS traffic from the new VPC CIDR to the Windows DCs.  See step 6 above.

### 2. Long timeout during update play ###

- this problem is caused by the new instance lacking a route to Internet to get packages for update.  A likely fix is ensuring there is a NAT Gateway in a public subnet and having a route for the Private Subnets to that NAT Gateway.  For example:
![Alt text](images/awx-route-table.png?raw=true "Route Table Example")