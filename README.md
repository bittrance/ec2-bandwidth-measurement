# AWS instance bandwidth measurement tool

WIP: This project is in development: your mileage is likely to be short.

This is a tool to measure bandwidth between arbitrary AWS EC2 instances. It
uses the excellent [iperf3](http://software.es.net/iperf/) to do the actual
measuring. The output is a iperf3 JSON stream with one record for each
instance type.

The tool starts up a (preferably large) instance with an `iperf3` server
and then starts up instances of the requested types, install `iperf3` and
runs tests against the server, producing a JSON report, finally terminating
the instance.

Instances are tagged so that you can attribute the cost in your billing.
Running a non-trivial list of hosts will typically cost tens of USD. Note that AWS EC2 minimum debit interval is 1 hour, so 

*Buyer beware*: measuring bandwidth well is *hard* when you have two hosts
and a crossed TP cable. AWS EC2 is a massive global infrastructure with
software defined networking, which makes it a fantasy land where essentially
anything is possible.

## Limitations

Currently, this tool uses the private address of started hosts, which
effectively means you need a VPN that is part of the security group that
you give to instances.

## Prerequisites

- You need a table of instance type/ami pairs like so:
    ```
    m1.small ami-a71f3fd4
    c1.medium ami-a71f3fd4
    m1.medium ami-a71f3fd4
    m3.medium ami-7c1d3d0f
    c3.large ami-7c1d3d0f
    m1.large ami-a71f3fd4
    m3.large ami-7c1d3d0f
    ```

- You need aws CLI installed and environment variables or similar with
  credentials that allows it to do `run-instances`, `terminate-instances`
  and `create-tags`.

- An ssh key installed in the region(s) you want to run instances in that
  have `sudo` access on the host.

- You can run the commands herein from your workstation, but since they
  use synchronous ssh sessions to start iperf3 servers and clients, you
  are probably better off running them in screen/tmux in a dedicated EC2
  instance.

### Find a relevant AMI for a region

You can use the AWS CLI to extract AMI ID:s.
```
aws ec2 describe-images --owners amazon --filters Name=architecture,Values=x86_64 Name=root-device-type,Values=instance-store Name=virtualization-type,Values=paravirtual | jq -r '.Images[] | select(.Name != null) | select(.Name | contains("amzn-ami")) | select(.Name | contains("minimal") | not) | .Name,.ImageId'
```

## Running
```
$ bin/run-measurements < instances-and-amis.list > bandwidth-$(date +"%Y-%m-%d").json
```

### Running single commands

```
bin/network-utilization-server -k bittrance
bin/network-utilization-client -k bittrance -t m1.xlarge -a ami-a71f3fd4 -i 10.1.2.3
```

If you want to start your instance in a VPC, you need to use the `-s` and give a subnet id in which to start the server or client instance:
```
bin/network-utilization-server -k bittrance -z eu-west-1a -s subnet-12345678 -a ami-ae0937c8
```
Currently, the instance will be added to the VPC default security group.

## Output

The result from a run is a JSON stream. If you want to further process the
data, you can ask `jq -s` to turn it back into an array.

A quick summary can be generated thus:
```bash
jq -s '.[]|{
  "type":.instance_type,"ami":.ami_id,
  "sent_mbps":(.end.sum_sent.bits_per_second/1048576),
  "retransmits":.end.sum_sent.retransmits,
  "recv_mbps":(.end.sum_received.bits_per_second/1048576),
  "avg_total_cpu":.end.cpu_utilization_percent.host_total
  }' < bandwidth-2017-04-13.json
```

# About AWS bandwidth

AWS bandwidth is policed on instance ingress, so if you measure TCP
`m1.small` -> `c3.8xlarge` you will get massive numbers and i you measure
TCP `c3.8xlarge` -> `m1.small` you will get puny numbers.

The small end of the instance type spectrum have problem generating
multi-gigabit TCP streams, so keep an eye on client CPU usage in the
reports.
