https://docs.aws.amazon.com/pt_br/vpc/latest/userguide/vpc-subnets-commands-example.html
https://docs.fortinet.com/document/fortigate-public-cloud/7.0.0/aws-administration-guide/506140/connecting-a-local-fortigate-to-an-aws-vpc-vpn

aws ec2 create-vpc --cidr-block 10.10.0.0/16 --tag-specification 'ResourceType=vpc,Tags=[{Key=Name,Value=AWS-vpc}]'
export vpcid01=<VPC-ID>

aws ec2 create-vpc --cidr-block 10.20.0.0/16 --tag-specification 'ResourceType=vpc,Tags=[{Key=Name,Value=ONPREMISE}]'
export vpcid02=<VPC-ID>



aws ec2 create-subnet --vpc-id $vpcid01 --cidr-block 10.10.0.0/24 --tag-specification 'ResourceType=subnet,Tags=[{Key=Name,Value=AWS-subnet}]'
export subnetid01=<SUBNET-ID>

aws ec2 create-subnet --vpc-id $vpcid02 --cidr-block 10.20.20.0/24 --tag-specification 'ResourceType=subnet,Tags=[{Key=Name,Value=ONPREMISE-external}]'
export subnetid02=<SUBNET-ID>

aws ec2 create-subnet --vpc-id $vpcid02 --cidr-block 10.20.30.0/24 --tag-specification 'ResourceType=subnet,Tags=[{Key=Name,Value=ONPREMISE-internal}]'
export subnetid03=<SUBNET-ID>



aws ec2 create-internet-gateway --tag-specification 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=AWS-igw}]'
export igwid01=<IGW-ID>

aws ec2 create-internet-gateway --tag-specification 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=ONPREMISE-igw}]'
export igwid02=<IGW-ID>

aws ec2 attach-internet-gateway --vpc-id $vpcid01 --internet-gateway-id $igwid01
aws ec2 attach-internet-gateway --vpc-id $vpcid02 --internet-gateway-id $igwid02

aws ec2 create-route-table --vpc-id $vpcid01 --tag-specification 'ResourceType=route-table,Tags=[{Key=Name,Value=AWS-rt}]'
export rtid01=<RT-ID>

aws ec2 create-route-table --vpc-id $vpcid02 --tag-specification 'ResourceType=route-table,Tags=[{Key=Name,Value=ONPREMISE-rt-external}]'
export rtid02=<RT-ID>

aws ec2 create-route-table --vpc-id $vpcid02 --tag-specification 'ResourceType=route-table,Tags=[{Key=Name,Value=ONPREMISE-rt-internal}]'
export rtid03=<RT-ID>

aws ec2 create-route --route-table-id $rtid01 --destination-cidr-block 0.0.0.0/0 --gateway-id $igwid01
aws ec2 create-route --route-table-id $rtid02 --destination-cidr-block 0.0.0.0/0 --gateway-id $igwid02
aws ec2 create-route --route-table-id $rtid03 --destination-cidr-block 0.0.0.0/0 --gateway-id $igwid02

aws ec2 describe-route-tables --route-table-id $rtid01
aws ec2 describe-route-tables --route-table-id $rtid02
aws ec2 describe-route-tables --route-table-id $rtid03

aws ec2 describe-subnets --filters "Name=vpc-id,Values=$vpcid01" --query "Subnets[*].{ID:subnetid,CIDR:CidrBlock}"
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$vpcid02" --query "Subnets[*].{ID:subnetid,CIDR:CidrBlock}"

aws ec2 associate-route-table --subnet-id $subnetid01 --route-table-id $rtid01
aws ec2 associate-route-table --subnet-id $subnetid02 --route-table-id $rtid02
aws ec2 associate-route-table --subnet-id $subnetid03 --route-table-id $rtid03

aws ec2 modify-subnet-attribute --subnet-id $subnetid01 --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $subnetid02 --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $subnetid03 --map-public-ip-on-launch

aws ec2 create-key-pair --key-name MyKeyPair --query "KeyMaterial" --output text > MyKeyPair.pem

chmod 400 MyKeyPair.pem

aws ec2 create-security-group --group-name AWS-sg --description "Security group for AWS" --vpc-id $vpcid01
export sgid01=<SG-ID>

aws ec2 create-security-group --group-name ONPREMISE-internal --description "Security group for ONPREMISE Internal" --vpc-id $vpcid02
export sgid02=<SG-ID>

aws ec2 authorize-security-group-ingress --group-id $sgid01 --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $sgid02 --protocol tcp --port 22 --cidr 0.0.0.0/0


aws ec2 run-instances --image-id ami-a4827dc9 --count 1 --instance-type t2.micro --key-name MyKeyPair --security-group-ids $sgid01 --subnet-id $subnetid01 --tag-specification 'ResourceType=instance,Tags=[{Key=Name,Value=AWS-ec2}]'
export instid01=<INST-ID>

aws ec2 run-instances --image-id ami-a4827dc9 --count 1 --instance-type t2.micro --key-name MyKeyPair --security-group-ids $sgid02 --subnet-id $subnetid03 --tag-specification 'ResourceType=instance,Tags=[{Key=Name,Value=ONPREMISE-ec2}]'
export instid02=<INST-ID>

aws ec2 describe-instances --instance-id $instid01
export pubip01=<PUB-IP>

aws ec2 describe-instances --instance-id $instid02
export pubip02=<PUB-IP>

ssh -i "MyKeyPair.pem" ec2-user@$pubip01
ssh -i "MyKeyPair.pem" ec2-user@$pubip02

-----------------------------------------

aws ec2 allocate-address --tag-specification 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=ONPREMISE-EIP}]'
export pubeip01=<PUB-IP>

----------------------------------------------

aws ec2 create-vpn-gateway --type ipsec.1
export vgwid=<VGW-ID>

aws ec2 attach-vpn-gateway --vpn-gateway-id $vgwid --vpc-id $vpcid01

aws ec2 create-customer-gateway --type ipsec.1 --public-ip $pubeip01 --bgp-asn 65000

export cgwid=<CGW-ID>

aws ec2 create-vpn-connection \
    --type ipsec.1 \
    --customer-gateway-id $cgwid \
    --vpn-gateway-id $vgwid \
    --tag-specification 'ResourceType=vpn-connection,Tags=[{Key=Name,Value=VPN-FGT}]' \
    --options "{\"StaticRoutesOnly\":true}"

-------------------------------------------------------------------------------------

config vpn ipsec phase1-interface

edit "awsphase1"

set interface "port1"

set keylife 28800

set peertype any

set proposal aes128-sha1

set dhgrp 2

set remote-gw 34.197.64.207

set psksecret sXvwsChnv_ASwI98ExivYOCEyZjtOsgA

set dpd-retryinterval 10

next

end

config vpn ipsec phase2-interface

edit "awsphase2"

set phase1name "awsphase1"

set proposal aes128-sha1

set dhgrp 2

set keylifeseconds 3600

next

end

config router static

edit 1

set dst 10.10.0.0 255.255.0.0

set device "awsphase1"

next

end

config firewall policy

edit 1

set srcintf "awsphase1"

set dstintf "port2"

set srcaddr "all"

set dstaddr "all"

set action accept

set schedule "always"

set service "ALL"

next

edit 2

set srcintf "port2"

set dstintf "awsphase1"

set srcaddr "all"

set dstaddr "all"

set action accept

set schedule "always"

set service "ALL"

next

end

diagnose vpn tunnel up awsphase2

-------------------------------------------------------------------------------------

aws ec2 terminate-instances --instance-ids $instid01

aws ec2 delete-security-group --group-id $sgid01

aws ec2 delete-subnet --subnet-id $subnetid01

aws ec2 delete-route-table --route-table-id $rtid01

aws ec2 detach-internet-gateway --internet-gateway-id $igwid01 --vpc-id $vpcid01

aws ec2 delete-internet-gateway --internet-gateway-id $igwid01

aws ec2 delete-vpc --vpc-id $vpcid01



aws ec2 describe-vpcs
