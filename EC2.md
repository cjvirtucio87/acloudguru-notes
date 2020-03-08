# Setup

## Profile

Need the following profile `"${HOME}/.aws/credentials"`:

```
[personal]
region = us-east-1
aws_secret_access_key = [REDACTED]
aws_access_key_id = [REDACTED]
ca_bundle = /path/to/ca/bundle
```

## ssh

When launching an instance, generate a keypair and save the `pem` file (this is the private key; it's generated only _once_, so keep it somewhere safe).

You can then `ssh` into your `ec2` instance with the ff. command:

```
ssh \
  -i '/path/to/your/pem/file' \
  "ec2-user@$(aws \
                --profile personal \
                ec2 \
                  describe-instances \
                  | jq \
                    -r \
                    '.Reservations[0].Instances[0].NetworkInterfaces[0].Association.PublicIp')"
```

## httpd

You turn your `ec2` instance into a webserver by installing and starting `httpd`:

```
sudo yum update -y
sudo yum install httpd -y
sudo service httpd start
```

Then tail its logs (to verify access) and `curl` it from another terminal:

```
# from ec2 instance
sudo tail -f /var/log/httpd/access_log

# from other terminal
curl \
  "ec2-user@$(aws \
                --profile personal \
                ec2 \
                  describe-instances \
                  | jq \
                    -r \
                    '.Reservations[0].Instances[0].NetworkInterfaces[0].Association.PublicIp')"
```

_tip: you can just run the `aws describe-instances | jq` command described above to grab the `PublicIp` and save it in an environment variable_

# Security Groups

## Revoking Inbound Rule

```
aws \
  --profile personal \
  ec2 describe-instances \
  | jq '.Reservations[0].Instances[0].SecurityGroups[0].GroupId' -r \
    | xargs -I {} aws \
      --profile personal \
      ec2 revoke-security-group-ingress \
      --group-id {} \
      --ip-permissions '[{"IpProtocol": "tcp", "FromPort": 80, "ToPort": 80, "IpRanges": [{"CidrIp": "0.0.0.0/0"}], "Ipv6Ranges": [{"CidrIpv6": "::/0"}]}]'
```

## Authorizing Inbound Rule

```
aws \
  --profile personal \
  ec2 describe-instances \
  | jq '.Reservations[0].Instances[0].SecurityGroups[0].GroupId' -r \
    | xargs -I {} aws \
      --profile personal \
      ec2 authorize-security-group-ingress \
      --group-id {} \
      --ip-permissions '[{"IpProtocol": "tcp", "FromPort": 80, "ToPort": 80, "IpRanges": [{"CidrIp": "0.0.0.0/0"}], "Ipv6Ranges": [{"CidrIpv6": "::/0"}]}]'
```

# Instances

## Key Pairs

To create a key pair:

```
aws ec2 create-key-pair --key-name <your key-pair name>
```

The json response will look like this:

```
{
    "KeyFingerprint": "...",
    "KeyMaterial": "...",
    "KeyName": <your key-pair name>,
    "KeyPairId": "..."
}
```

Save the content of `KeyMaterial` in a file in your `.ssh` folder. Lock down the permissions:

```
chmod 0600 ~/.ssh/<your private key with the KeyMaterial>
```

## Creating

```
aws ec2 run-instances \
  --image-id <id for Amazon Linux 2 AMI (HVM), SSD Volume Type> \
  --count 1 \
  --key-name <your key-pair name> \
  --security-groups web-dmz \
  --block-device-mappings "$(cat templates/amazon_linux_2_hv2_ssd_volume_type/block_device_mappings.json)"
```

# Storage

## EBS-backed vs Instance Store-backed

There's a summary [here](https://medium.com/awesome-cloud/aws-difference-between-ebs-and-instance-store-f030c4407387).

## Transferring EBS volume to a different AZ

1. Create a snapshot of the EBS volume in a different AZ from the volume.
1. Create an image from the snapshot.
1. Launch an instance from the image (AMI) that you created.

You now have a running instance with a volume in a different AZ.

## Encryption

### Encrypt an unencrypted volume

1. Create a snapshot of the volume.
1. Copy the snasphot (with `Encrypt this snapshot` checked)
1. Create an image from the newly-encrypted snapshot.
1. Launch a new instance with the newly-created AMI.

You now have a running instance with an encrypted volume.

### New instance from image with encrypted volume must also have its volume encrypted

You can't take an image that has its volume encrypted and launch a new instance with that volume set to _unencrypted_.

# CloudWatch

## Metrics monitored

* Compute
* Storage
* Content Delivery
* Host-level Metrics

## vs. CloudTrail

CloudTrail tracks user and resource activity through the console and through the API.

CloudTrail is for auditing; Cloud _Watch_ is for performance metrics.

