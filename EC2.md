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
